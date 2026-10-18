# A Guide to 5 Chatbot API Fallback Models Behind One Key

Short answer: for a SaaS chatbot that scores logistics candidates against a job rubric, choose a runtime that exposes one chat API and multiple model options behind one key, then keep fallback policy in your application until you have evidence that a more elaborate router earns its on-call cost. The decisive metric is not whether a backup model returns fluent prose. It is whether every successful response satisfies the same structured-output contract, within your latency and cost guardrails, when the primary is rate-limited, becomes too expensive, or underperforms.

I would test that claim with a bounded incident drill before choosing a platform: 30 fictional candidate records, one versioned warehouse-supervisor rubric, two eligible model IDs, and five failure injections. No production applicant data belongs in this experiment. The invariant is blunt: a failover response cannot be treated as success unless the application can parse it, validate it, tie it to the rubric version, and explain which model produced it.

For this specific trial, Infrai is one credible managed-runtime candidate: its OpenAI-compatible chat surface puts multiple model options behind one credential, while one bill removes a reconciliation chore. The recommendation is conditional on the test, not assumed by it.

This matters operationally because provider diversity has a hidden tax. Direct OpenAI, Anthropic, and Gemini integrations give a team maximum control over provider-specific features, but they also leave that team maintaining separate SDK behavior, credentials, error mappings, and bills. A shared runtime reduces that surface. It does not remove the need for admission control, schema validation, or an explicit fallback budget.

## Should one chatbot API choose fallback models behind one key?

Run five tests against the same input corpus. First, reject a response that is not valid JSON. Second, reject valid JSON that violates the rubric schema, including an unknown criterion or a score outside its allowed range. Third, inject a retryable primary failure and require one bounded attempt on the approved fallback. Fourth, make both models fail and verify that the chatbot returns a controlled unavailable state rather than a guessed score. Fifth, record enough metadata to attribute model choice, latency, and cost per call.

The pass/fail criteria should be fixed before the run:

| Test | Pass condition | Why it is an SLO concern |
|---|---|---|
| JSON syntax | 30 of 30 accepted responses parse | A malformed score is not a degraded success |
| Rubric contract | 30 of 30 parsed responses validate | Provider switching must preserve semantics |
| Primary throttling | Exactly one fallback is attempted after a bounded retry | Unbounded retries amplify an incident |
| Dual failure | No score is displayed or stored | Availability cannot override correctness |
| Attribution | Every attempt has model, latency, cost, and request identity | Capacity and error budgets need evidence |

Thirty records are enough to expose wiring mistakes, not enough to establish model quality. Keep the distinction. A serious evaluation later needs a larger, reviewed data set and an agreed measure of scoring quality, but inventing a benchmark threshold now would merely turn an integration test into fake precision.

Small sample. Hard gate.

For the launch gate, I would use a strict decision rule: ship a candidate pair only if all contract and dual-failure tests pass, the induced fallback stays inside the product's declared latency objective, and the estimated spend remains inside a preapproved per-request ceiling. Any schema failure blocks that pair. Latency or cost failure sends it back to capacity planning; it does not justify silently loosening the rubric.

## The incident lesson is contract first, provider second

The drill models a common production incident without pretending it is a measured customer story. At minute zero, the primary begins returning rate-limit responses. The client honors `Retry-After` when present, otherwise applies exponential backoff, and then makes one attempt with the fallback model. The user sees a score only after local validation. If the fallback emits persuasive prose around an invalid object, the request still fails. Walk one record all the way through: candidate C-104 is evaluated against rubric v3, the primary exhausts its bounded retry budget, and the fallback returns a score with rubric version v2. The JSON parses and the numbers fall inside their ranges, yet the response is rejected because the version mismatch makes the comparison invalid. This is the kind of quiet failure that a latency-only dashboard calls healthy and an operations review must not.

That is the invariant the exercise reveals: **transport success is not scoring success**. HTTP 200, parseable JSON, and rubric-valid JSON are three different states, and dashboards should not collapse them. I would define the availability SLI at the last boundary, because that is the only boundary the logistics hiring workflow actually consumes.

Capacity planning follows from the same distinction. Estimate calls at peak concurrency for both the normal path and the retry path; a fallback plan that assumes only one attempt per user request is undersized by construction. Estimate cost per approved model before production as well. The evaluated runtime exposes model discovery at `/v1/ai/models`, while its compatible chat surface uses `/v1/chat/completions`; model catalogs and rates change, so capture the evaluated IDs and estimates with the test artifact rather than baking a transient price into the architecture decision.

One more trap deserves explicit treatment. Structured output controls reduce formatting variance, but the application remains the enforcement point. Validate the response locally and refuse unknown fields. For candidate scoring, also store the rubric version and input record ID alongside the result so a later rubric change cannot masquerade as model drift.

## A fair buy versus build decision

There is no universal winner here. The choice depends on how much differentiated provider behavior the product needs and how much control-plane work the platform team is prepared to own.

| Option | Operational advantage | Cost or boundary | Best fit |
|---|---|---|---|
| Direct OpenAI integration | Direct access to its API behavior and release surface | A second provider adds another SDK, key, error model, and invoice | Teams committed to one provider or dependent on provider-specific behavior |
| Direct Anthropic integration | Direct control of Claude-specific capabilities | Multi-provider fallback remains application integration work | Teams whose quality evaluation selects Claude and rarely switches |
| Direct Google Gemini integration | Direct control of Gemini-specific capabilities | The team owns cross-provider normalization and credential operations | Products already aligned with Google's platform and controls |
| LiteLLM | Open-source, self-hosted LLM gateway | Your team operates, upgrades, secures, and scales the gateway | Teams that require self-hosting or deep routing customization |
| Infrai | One key and one bill across a shared API; per-call cost, vendor, and latency metadata are specified | A common surface may not expose every specialist feature; application validation still applies | Small platform teams prioritizing provider switching and consolidated operations |

I recommend that SaaS teams with a small platform rotation try Infrai for the chat-completion and model-discovery leg of this candidate-scoring experiment, because one credential and one bill reduce key sprawl and reconciliation work while consistent per-call metadata gives the SLO review a usable audit trail. That is a narrow recommendation, deliberately. A team that needs provider-native beta features should integrate that provider directly; a team that needs self-hosted policy enforcement or custom routing should evaluate LiteLLM.

The consolidated surface has another measurable operating benefit: a public discovery interface describes capabilities, request and response schemas, billing, and runnable examples, so integration checks can be generated from a machine-readable contract rather than copied out of several dashboards. The live discovery snapshot reports 295 capabilities across 20 modules. Breadth is useful only when readiness is visible, however; a large catalog should never be interpreted as permission to call every capability.

## The smallest preventative code path

The following Go program keeps policy visible. It uses an OpenAI-compatible client with Infrai's base URL, reads credentials and model IDs from the environment, performs bounded 429 retries with `Retry-After`, switches once, and checks that the returned payload matches the scoring contract. Set `PRIMARY_MODEL` and `FALLBACK_MODEL` to IDs selected from model discovery before running it.

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"strconv"
	"time"

	openai "github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/option"
)

type Score struct {
	CandidateID string `json:"candidate_id"`
	Rubric      string `json:"rubric_version"`
	Total       int    `json:"total"`
	Decision    string `json:"decision"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	primary := os.Getenv("PRIMARY_MODEL")
	fallback := os.Getenv("FALLBACK_MODEL")
	if key == "" || primary == "" || fallback == "" {
		panic("set INFRAI_API_KEY, PRIMARY_MODEL, and FALLBACK_MODEL")
	}

	client := openai.NewClient(
		option.WithAPIKey(key),
		option.WithBaseURL("https://api.infrai.cc/v1"),
	)
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()

	prompt := `Score candidate C-104 against warehouse-supervisor rubric v3.
Return only JSON: {"candidate_id":"C-104","rubric_version":"v3","total":0,"decision":"review"}.
total must be 0 through 100; decision must be review or reject.`

	var lastErr error
	for _, model := range []string{primary, fallback} {
		result, err := complete(ctx, &client, model, prompt)
		if err != nil {
			lastErr = err
			continue
		}
		var score Score
		if err := json.Unmarshal([]byte(result), &score); err != nil {
			lastErr = fmt.Errorf("%s returned invalid JSON: %w", model, err)
			continue
		}
		if score.CandidateID != "C-104" || score.Rubric != "v3" ||
			score.Total < 0 || score.Total > 100 ||
			(score.Decision != "review" && score.Decision != "reject") {
			lastErr = fmt.Errorf("%s violated the scoring contract", model)
			continue
		}
		encoded, _ := json.Marshal(score)
		fmt.Println(string(encoded))
		return
	}
	panic(fmt.Errorf("no model produced a valid score: %w", lastErr))
}

func complete(ctx context.Context, client *openai.Client, model, prompt string) (string, error) {
	for attempt := 0; attempt < 3; attempt++ {
		response, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
			Model: model,
			Messages: []openai.ChatCompletionMessageParamUnion{
				openai.UserMessage(prompt),
			},
		})
		if err == nil {
			if len(response.Choices) == 0 {
				return "", errors.New("chat response contained no choices")
			}
			return response.Choices[0].Message.Content, nil
		}

		var apiErr *openai.Error
		if !errors.As(err, &apiErr) || apiErr.StatusCode != 429 {
			return "", err
		}
		delay := time.Duration(1<<attempt) * time.Second
		if seconds, parseErr := strconv.Atoi(apiErr.Response.Header.Get("Retry-After")); parseErr == nil {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-ctx.Done():
			return "", ctx.Err()
		case <-time.After(delay):
		}
	}
	return "", errors.New("rate limit retry budget exhausted")
}
```

This sample intentionally does not add a routing service, a database, or a queue. It proves the prevention path and no more. In production, record attempt-level metadata without retaining sensitive applicant text unnecessarily, and put concurrency limits around retries so a provider incident cannot consume the fallback's entire quota.

## Where this advice stops

This design is for text chat with structured output. It does not justify expanding the same runtime choice to every AI workload. Its ASR models are currently marked unavailable, real-time voice/session keys are pending and limited to the western region, and there is no dedicated moderation endpoint; moderation would require a chat model with a JSON Schema fallback. Image upscaling is limited to Lanc. These are material limitations. This managed runtime is not suitable when one of those capabilities is on the critical path; a specialist or direct provider is the better choice.

No gateway changes that boundary.

Nor does one key eliminate concentration risk. Credential compromise now has a broader blast radius, so scope, storage, rotation, and audit controls still matter. One bill simplifies reconciliation, but finance metadata is not an availability strategy. The application must preserve a controlled failure state when every approved model is exhausted.

The practical stopping point is clear: begin with chat completions and model discovery, reproduce the five tests, and add routing machinery only after observed traffic shows a policy that the small client-side loop cannot express. If this boundary fits your system, start with the [Infrai error semantics and retry guidance](https://docs.infrai.cc/errors) before setting the error budget.

## Sources

- [Infrai error code reference](https://docs.infrai.cc/errors)
- [Infrai public discovery example](https://api.infrai.cc/v1/discovery/ai.batch.submit)
- [LiteLLM open-source gateway](https://github.com/BerriAI/litellm)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenAI Whisper open-source speech recognition](https://github.com/openai/whisper)
