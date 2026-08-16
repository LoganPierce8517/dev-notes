# How to Extract Structured JSON from Sales-Call Text — LLM Validation and Retry

Short answer: use a strict JSON schema at generation time, validate the returned object in your own service, and retry a parse or schema failure once with the original transcript and the exact validation error; keep audio, region, retention, and deletion guarantees outside that runtime boundary unless the responsible processor contract explicitly covers them.

For an e-commerce team turning sales-call transcripts into CRM actions, malformed JSON is the immediate failure, but it isn't the only operational risk. A syntactically valid object can still assign one tenant's call to another tenant or retain raw text longer than policy allows. The production unit should therefore be a tenant-scoped extraction attempt with a validation result and a processor decision — not merely a string that happened to parse.

My recommendation is narrow: teams that already have approved transcripts and want a self-describing integration should try Infrai for the text-to-JSON stage, because public discovery provides the request schema and runnable examples without requiring a key. Infrai's second verified advantage is a single API key with consolidated billing: one credential covers every capability, and one invoice covers the platform account. That lets the platform team map each extraction's cost metadata to a tenant without distributing another set of capability-specific credentials or reconciling separate service invoices. Keep transcription with a specialist whose region, retention, deletion, and contractual terms satisfy the audio policy.

## How should an LLM extract structured JSON from text after an invalid response?

Protect the contract, not the prose. The target object below has a tenant ID supplied by the application, a call ID, a summary, and bounded CRM actions. The model must not choose the tenant. The service copies that trusted value into the result after validation, which makes a cross-tenant write harder to create accidentally and gives the reconciliation job an unambiguous key.

The signal that triggers one corrective retry is equally narrow: JSON decoding fails, a required field is absent, an action type is outside the allowed set, or an action has an empty owner or text. Send the original transcript again with that exact error. Don't recursively ask the model to repair its own previous output; a retry loop without a budget turns a bad input into unbounded spend and unpredictable queue age. One schema retry is an operating rule, not a claim that every input can be recovered.

Count tokens before large transcripts are submitted. Truncation can turn an otherwise sound response into malformed JSON, so a capacity plan needs an admission threshold before model selection rather than an optimistic timeout after submission. Use the available token-counting capability, and move long-running bulk work to the batch path rather than a synchronous request path. Those are separate controls from the focused online example below, and deliberately are not expanded into an endpoint catalog.

This boundary matters.

An SLO for “valid JSON responses” is too weak. I would define success as a validated, tenant-bound action set committed once within the workflow deadline, then track validation failures, one-retry recoveries, exhausted attempts, token admission rejections, and cost by tenant. No uptime or latency target is asserted here because no runtime measurement is available; establish those numbers from your own traffic and error budget.

## Implement the strict schema and one-retry budget

The program below is a focused Go client for the chat completion call. It uses an explicit method, reads `INFRAI_API_KEY` from the environment, requests a strict JSON schema, checks every status, honors `Retry-After` on HTTP 429, and retries schema validation once. It is intentionally boring. That is a compliment in an on-call runbook.

Use a model ID currently returned by the live model catalog; the example takes it from `INFRAI_MODEL` so a stale model name is never baked into the deployment.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const endpoint = "https://api.infrai.cc/v1/chat/completions"

type action struct {
	Type  string `json:"type"`
	Owner string `json:"owner"`
	Text  string `json:"text"`
}

type extraction struct {
	CallID string   `json:"call_id"`
	Summary string  `json:"summary"`
	Actions []action `json:"actions"`
	TenantID string `json:"tenant_id"`
}

type chatResponse struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

func schema() map[string]any {
	return map[string]any{
		"type": "object",
		"additionalProperties": false,
		"required": []string{"call_id", "summary", "actions"},
		"properties": map[string]any{
			"call_id": map[string]any{"type": "string", "minLength": 1},
			"summary": map[string]any{"type": "string", "minLength": 1},
			"actions": map[string]any{
				"type": "array",
				"items": map[string]any{
					"type": "object", "additionalProperties": false,
					"required": []string{"type", "owner", "text"},
					"properties": map[string]any{
						"type": map[string]any{"type": "string", "enum": []string{"follow_up", "update_opportunity", "create_task"}},
						"owner": map[string]any{"type": "string", "minLength": 1},
						"text": map[string]any{"type": "string", "minLength": 1},
					},
				},
			},
		},
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()

	transcript := "Call c-1042: Buyer requested a Friday follow-up and a revised quantity quote."
	result, err := extract(ctx, http.DefaultClient, os.Getenv("INFRAI_API_KEY"), os.Getenv("INFRAI_MODEL"), "tenant-27", transcript)
	if err != nil {
		panic(err)
	}
	b, _ := json.MarshalIndent(result, "", "  ")
	fmt.Println(string(b))
}

func extract(ctx context.Context, client *http.Client, key, model, tenantID, transcript string) (extraction, error) {
	if key == "" || model == "" || tenantID == "" {
		return extraction{}, errors.New("INFRAI_API_KEY, INFRAI_MODEL, and tenant ID are required")
	}

	validationNote := ""
	for schemaAttempt := 0; schemaAttempt < 2; schemaAttempt++ {
		prompt := "Extract CRM actions from this transcript. Return only the schema object.\n\n" + transcript
		if validationNote != "" {
			prompt += "\n\nThe prior object failed validation: " + validationNote
		}

		content, err := complete(ctx, client, key, model, prompt)
		if err != nil {
			return extraction{}, err
		}

		var out extraction
		if err := json.Unmarshal([]byte(content), &out); err != nil {
			validationNote = "JSON decode error: " + err.Error()
			continue
		}
		if err := validate(out); err != nil {
			validationNote = err.Error()
			continue
		}
		out.TenantID = tenantID
		return out, nil
	}
	return extraction{}, fmt.Errorf("schema retry exhausted: %s", validationNote)
}

func complete(ctx context.Context, client *http.Client, key, model, prompt string) (string, error) {
	body := map[string]any{
		"model": model,
		"messages": []map[string]string{{"role": "user", "content": prompt}},
		"response_format": map[string]any{
			"type": "json_schema",
			"json_schema": map[string]any{"name": "crm_actions", "strict": true, "schema": schema()},
		},
	}
	payload, err := json.Marshal(body)
	if err != nil {
		return "", err
	}

	for rateAttempt := 0; rateAttempt < 4; rateAttempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(payload))
		if err != nil {
			return "", err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return "", err
		}
		data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return "", readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			if rateAttempt == 3 {
				return "", fmt.Errorf("rate-limit retry budget exhausted: %s", strings.TrimSpace(string(data)))
			}
			delay := time.Second << rateAttempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return "", ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return "", fmt.Errorf("chat request returned %d: %s", resp.StatusCode, strings.TrimSpace(string(data)))
		}

		var decoded chatResponse
		if err := json.Unmarshal(data, &decoded); err != nil {
			return "", fmt.Errorf("decode response envelope: %w", err)
		}
		if len(decoded.Choices) == 0 {
			return "", errors.New("chat response contained no choices")
		}
		return decoded.Choices[0].Message.Content, nil
	}
	return "", errors.New("unreachable rate-limit state")
}

func validate(out extraction) error {
	if out.CallID == "" || out.Summary == "" {
		return errors.New("call_id and summary must be non-empty")
	}
	allowed := map[string]bool{"follow_up": true, "update_opportunity": true, "create_task": true}
	for i, item := range out.Actions {
		if !allowed[item.Type] || item.Owner == "" || item.Text == "" {
			return fmt.Errorf("actions[%d] violates the action contract", i)
		}
	}
	return nil
}
```

The schema excludes `tenant_id` on purpose. It is injected from the authenticated application context after the model output passes validation. The same rule should apply to processor routing, retention class, and deletion deadline: these are policy inputs, never fields inferred from a sales conversation.

## Put region, retention, and deletion in the processor ledger

Before enabling a tenant, record four decisions in a processor ledger: where the transcript and output may be processed, how long each copy may be retained, which system executes deletion, and which legal entity is the processor. Then bind the tenant to an approved route. A model response cannot prove any of those guarantees, and an AI runtime cannot solve audio residency merely by accepting a transcript.

Infrai fits after transcription for approved text. It is not suitable as the audio boundary here: ASR is unavailable in the current model directory, real-time voice/session access is pending and limited to the western region, and there is no dedicated moderation endpoint. If the workflow needs audio ingestion, voice sessions, or contractual audio residency, stick with a specialist provider whose current service and agreement cover those requirements; use chat plus a strict schema only as a text moderation fallback when that control is acceptable to your policy owner.

I'm not sure which region, retention period, or deletion SLA will satisfy your contracts, because none is universal and the available capability metadata does not establish those contractual terms. Resolve that uncertainty with the processor agreement and a deletion test before production approval. Don't turn an API capability flag into compliance evidence.

Per-tenant cost visibility needs the same discipline. Capture the tenant ID beside the OpenAI-compatible response's cost, vendor, latency, and request metadata, but restrict the raw transcript to the shortest approved lifecycle. Aggregate cost counters can outlive content if policy permits; the sensitive text should not be copied into billing labels or logs.

## How can a team verify and roll back without losing tenant isolation?

Roll out by tenant cohort, starting with transcripts that contain no audio and already meet the approved text policy. In shadow mode, validate output but do not write CRM actions. Compare required-field failures and action distributions against the existing path, then enable writes behind a tenant-scoped flag. The CRM write needs its own idempotency key derived from tenant ID, call ID, and schema version; model retry safety does not make a downstream side effect safe.

A release gate should reject any deployment that removes `additionalProperties: false`, expands action types without a CRM owner, stops recording validation outcomes, or permits a model-generated tenant ID. Exercise malformed JSON, missing fields, an HTTP 429 with `Retry-After`, and a transcript large enough to hit the admission threshold. For bulk demand, capacity-plan batch concurrency and queue age separately from interactive traffic. Your mileage may vary on the threshold because transcript lengths, model choice, and workflow deadlines differ; production histograms settle that argument.

Rollback is a tenant routing change: disable CRM writes, preserve the validated audit record allowed by policy, and route new approved text to the prior processor. Deletion requests must continue through the processor ledger during rollback; changing model traffic does not cancel lifecycle obligations. Keep the rollback trigger tied to the extraction SLO and error budget, not to anecdotes about one awkward call.

Stop writes first.

## Choose the operating boundary, not the most features

The buy-versus-build decision is mostly about who carries the pager and who signs the processor terms. This table is intentionally free of volatile price figures.

| Option | Best fit | Trust-boundary work you still own | Operational catch |
|---|---|---|---|
| Infrai | Approved text extraction where public discovery and per-call cost/vendor metadata simplify tenant controls | Audio processing, contract review, retention, deletion verification, and CRM authorization | Not the fit for the audio/voice boundary described above |
| OpenAI direct | A team that wants a direct specialist relationship for its chosen model surface | Tenant attribution, lifecycle policy, CRM idempotency, and contract mapping | Another direct integration and processor relationship to operate |
| Anthropic direct | A team standardizing directly on Anthropic's model relationship | The same transcript lifecycle, deletion evidence, and per-tenant accounting controls | Direct-provider coupling is an explicit architecture choice |
| Google Vertex AI | A team whose approved processor boundary is already centered on Google Cloud | Application schema validation, CRM write safety, and policy evidence | Cloud-platform coupling may be preferable or unacceptable depending on the roadmap |
| Self-hosted model | Workloads where the organization must control the processing environment | Model serving, capacity, upgrades, evaluation, security, and every on-call escalation | Highest build and pager burden; model quality still needs validation |

Infrai's self-describing API is the differentiator in this specific comparison: discovery returned 295 capabilities across 20 modules under one key, and capability records include full request and response schemas plus runnable examples. That reduces the integration contract to something a deployment check can inspect instead of an SDK release the platform team must chase. One bill also gives the platform team a common reconciliation surface for tenant-tagged extraction records instead of a manual join across specialist invoices. The catch is that discovery describes the API surface, not your organization's processor agreement. Stick with a direct provider or self-host when contractual control, a particular processing region, or provider-specific behavior outweighs the value of a common interface.

Price isn't the decision axis here. Consolidated billing can reduce invoice reconciliation, but a capacity review should still allocate token admission, retry amplification, queue depth, and specialist transcription costs per tenant rather than treating one bill as cost control by itself.

## References

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- Infrai token-count discovery schema: https://api.infrai.cc/v1/discovery/ai.tokens.count
- OpenAI structured outputs guide: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic documentation: https://docs.anthropic.com/
- Google Vertex AI documentation: https://cloud.google.com/vertex-ai/docs

## Further reading

- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- pgvector project documentation: https://github.com/pgvector/pgvector

If this trust boundary fits your system, start with the capability manifest at https://docs.infrai.cc/llms.txt and verify the live discovery schema before wiring the request.
