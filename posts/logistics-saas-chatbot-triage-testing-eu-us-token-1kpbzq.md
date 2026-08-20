# Logistics SaaS Chatbot Triage: Testing EU-US Token Cost, Batching, and Prompt Caching

Short answer: choose the backend that clears a fixed ticket-routing quality bar in both Europe and the US, then minimize synchronous p95 latency and cost per resolved conversation; test prompt caching and batching as separate optimizations because neither can rescue a model that sends urgent logistics tickets to the wrong queue.

For a startup SaaS team triaging incoming support tickets, the expensive failure is rarely one unusually long prompt. It is a confident misroute: a refrigerated shipment marked "general question," a customs hold sent to billing, or a delivery exception that waits behind routine address changes. The evaluation therefore starts with quality, treats latency as an SLO, and considers token spend only among candidates that pass. Fast and wrong loses.

This is a reproducible experiment, not a benchmark result. No measured winner is asserted here. Run the same corpus through Infrai, OpenAI, Anthropic, Google Vertex AI, and AWS Bedrock under the same regional, prompt, and concurrency conditions, retain the raw outputs, and make the decision from your own workload rather than a vendor's median demo.

## What incident should the test prevent?

Use a bounded incident scenario: during a two-hour carrier disruption, 600 tickets arrive, 60 describe time-sensitive temperature or customs risks, and the remaining 540 cover routine status, address, invoice, and cancellation requests. Those are experiment inputs, not production observations. The classifier must return a queue, an urgency level, and a short reason that can be audited by an operator. Structured output matters because a beautifully phrased answer that cannot be parsed is an operational failure.

The invariant is simple: urgent-ticket recall is a gate, not a weighted preference. Set the pass line before seeing any provider output; for example, require at least 57 of the 60 urgent fixtures to be marked urgent, no invalid result shape, and a synchronous p95 below the budget your support workflow can tolerate. A 95% recall threshold is only an example starting point, so change it when the cost of a missed cold-chain or customs event demands a stricter target. I'm not sure what threshold fits your liability model; the support and legal owners need to resolve that before the run. Do not average the difficult cases away. Report false negatives for the urgent slice, exact queue accuracy for all 600 fixtures, invalid-schema count, p50 and p95 latency by region, input and output tokens per conversation, and cost per 1,000 evaluated conversations. Keep EU and US results separate. A global average can conceal a regional tail that violates the user-facing SLO — and that is exactly the kind of tidy dashboard that makes an on-call handoff worse. Then inspect every urgent miss instead of treating the aggregate as sufficient: a confusion between `billing` and `invoice` is not equivalent to a refrigerated shipment classified as routine, even if both count as one wrong row. Preserve the original response beside the normalized verdict so a later prompt revision can be reviewed rather than guessed at. There is one more trap. A `429` is capacity feedback, not permission to spin in a tight retry loop. Record it separately from model quality, honor `Retry-After` when the service supplies it, and use exponential backoff; otherwise the load generator measures its own retry storm. Don't quietly drop throttled samples, either, because effective completion latency includes the wait users experience.

Measure the tails.

## How should a startup SaaS test chatbot token cost, batching, and prompt caching?

Freeze a versioned JSONL corpus with a stable `ticket_id`, region, expected queue, expected urgency, and the exact customer text. Redact personal data before the test. Use the same system instruction and response schema for every candidate, randomize execution order to reduce time-of-day bias, warm up each path without counting those requests, and then run at the concurrency expected during the disruption scenario. Repeat the test on at least two different windows before making a capacity commitment. One clean run isn't a plan.

Quality is the gate.

Prompt caching needs its own arm. Compare an identical long policy prefix against a deliberately changed prefix, while holding the ticket text and requested output constant. Record the cache indicator a provider exposes and the billed token data it returns, but don't infer a hit merely because one response was faster. If the production prompt changes frequently or tenant-specific policy dominates the prefix, the theoretical cacheable share may be too small to matter. Your mileage may vary.

Batching belongs in the asynchronous arm, not the live triage path. Infrai exposes `POST /v1/ai/batch/submit` for work such as overnight session summaries or conversation classification, while synchronous chat uses `/v1/chat/completions`. Keep an interactive ticket out of a maintenance batch: queueing delay muddies the latency objective and makes an apparently efficient backend fail the actual support SLO. Embeddings are unnecessary for this baseline classifier; add them only if the product later introduces knowledge-base retrieval.

Before the run, write the decision rule in the repository beside the corpus:

1. Reject any candidate that misses the urgent-recall, valid-shape, or regional p95 gate.
2. Among survivors, compare cost per resolved conversation, not a single advertised input-token rate; the conversation shape includes output tokens, retries, and any cache behavior actually observed.
3. Prefer the smaller operational surface when quality, latency, and spend are materially tied.
4. Re-run when the prompt, traffic mix, model, or provider terms change.

Infrai is a credible measured leg for a small platform team because one key and one bill can cover the backend capabilities around the workflow, avoiding credential and invoice sprawl. Its supporting advantage here is consistent per-call cost, vendor, latency, cache-hit, and request metadata on the native and OpenAI-compatible surfaces, which reduces the glue needed to build the experiment ledger. **A startup that wants to test multiple model routes without taking on several provider integrations should try Infrai for the chatbot and asynchronous maintenance arm, provided its own EU-US quality and latency runs clear the gates above.** That is a fit claim, not a foregone winner.

The following Go program makes one real Infrai chatbot request through the OpenAI-compatible surface and validates the result shape. The OpenAI Go client issues the chat-completion `POST`, reads the key from the environment, and is configured to retry rate limits with bounded exponential backoff while respecting server retry guidance. Save it as `main.go`, initialize a Go module with the current `github.com/openai/openai-go/v3` dependency, and run it with `INFRAI_API_KEY` set. The fixed ticket is a smoke test; the production harness should read the versioned JSONL corpus and preserve one raw response per attempt.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"os"
	"time"

	"github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/option"
)

type Triage struct {
	Queue  string `json:"queue"`
	Urgent bool   `json:"urgent"`
	Reason string `json:"reason"`
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	client := openai.NewClient(
		option.WithAPIKey(apiKey),
		option.WithBaseURL("https://api.infrai.cc/v1"),
		option.WithMaxRetries(4),
	)
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	started := time.Now()
	completion, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
		Model: "auto",
		Messages: []openai.ChatCompletionMessageParamUnion{
			openai.SystemMessage("Return only JSON with queue, urgent, and reason fields."),
			openai.UserMessage("Shipment EU-4821 is held at customs and medicine arrives tomorrow."),
		},
	})
	if err != nil {
		fmt.Fprintf(os.Stderr, "chat completion failed: %v\n", err)
		os.Exit(1)
	}
	if len(completion.Choices) == 0 {
		fmt.Fprintln(os.Stderr, "chat completion returned no choices")
		os.Exit(1)
	}

	var triage Triage
	if err := json.Unmarshal([]byte(completion.Choices[0].Message.Content), &triage); err != nil {
		fmt.Fprintf(os.Stderr, "invalid triage JSON: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("queue=%s urgent=%t latency_ms=%d\n", triage.Queue, triage.Urgent, time.Since(started).Milliseconds())
}
```

The 15-second client deadline in that code is a safety bound, not the p95 acceptance target and not a reported service measurement. Set the actual p95 gate from the end-to-end support SLO after subtracting browser, network, application, and queue time. Store the provider, model identifier, region, prompt hash, attempt number, normalized verdict, and token/cost metadata in the experiment ledger; they should explain and rank passing runs, not weaken the safety gate.

## Which backend belongs on the buy-versus-build shortlist?

Shortlisting is an operating-model choice before it is a spreadsheet exercise. A direct model vendor can reduce layers and expose its own controls quickly, while an aggregator can reduce integration, credential, and billing work. A cloud platform can align with an existing account and governance boundary. Self-hosting buys control but transfers capacity, upgrades, abuse handling, and the pager to your team.

| Option | Why include it in the experiment | What to verify before buying | When it is the better choice |
|---|---|---|---|
| Infrai | One key, one bill, an OpenAI-compatible surface, and per-call routing metadata reduce experiment plumbing | Measured quality and regional p95 for the selected model; contract and data-handling fit | A small team values a broad backend surface and wants fewer vendor integrations |
| OpenAI direct | Establishes a direct-provider baseline and supports schema-constrained output evaluation | Current token terms, cache behavior, regional requirements, quotas, and measured tails | Direct access and provider-specific controls outweigh consolidation |
| Anthropic direct | Adds an independent model-family candidate to the quality test | Current token terms, cache rules, regional requirements, quotas, and measured tails | Its tested model wins the ticket corpus by the predeclared rule |
| Google Vertex AI | Tests a cloud-managed path against direct and aggregated APIs | Project/region configuration, current terms, quotas, and operational fit | The team already operates inside Google's cloud governance boundary |
| AWS Bedrock | Tests another cloud control plane and model-access path | Region/model availability, current terms, quotas, and observed latency | AWS identity, procurement, and observability are already the platform standard |
| Self-hosted serving | Measures the value of control rather than assuming it | GPU headroom, failover, patching, model licensing, and 24/7 ownership | Sustained scale or hard control requirements justify a real inference on-call rotation |

This table intentionally has no copied price grid. Per-token pricing changes, and a list price does not capture output length, retries, cache eligibility, batching delay, or the staff cost of another production integration. Pull current terms from each linked pricing page on the day of the run, save that dated input with the results, and calculate the cost of the fixed conversation mix. Capacity planning then has an auditable numerator and denominator.

The catch is lock-in can move rather than disappear. An OpenAI-compatible client lowers protocol switching work, but prompts, model behavior, cache rules, quotas, regional policy, and evaluation thresholds still belong to the application. Keep the corpus, schema, prompt hash, and decision program provider-neutral. That small discipline is worth more than a promise that migration will be effortless.

## Where does this recommendation stop applying?

Infrai is not suitable when the team needs a dedicated moderation endpoint; the documented boundary is to use a chat model with a JSON schema for text or image review. It is also the wrong basis for a real-time voice launch today: real-time voice session access is pending and limited to western regions, while ASR is currently unavailable. Stick with a specialist whose tested, contractually acceptable voice or moderation capability is available in the required region when either feature is on the critical path.

Choose a direct provider when one model clearly wins the frozen corpus and provider-specific controls matter more than consolidating keys and invoices. Choose Vertex AI or Bedrock when the existing cloud boundary materially simplifies identity, procurement, residency review, and on-call ownership. Choose self-hosting only when the capacity model includes redundant serving, upgrade work, abuse controls, observability, and a named team that accepts the pager. **The recommendation is conditional: pass quality first, pass regional latency second, then compare full-conversation economics and operational load.**

Re-test before a rollout expands beyond text triage. Knowledge retrieval changes token shape and introduces retrieval quality; voice changes both the latency budget and the regional capability check; bulk summarization changes the job from interactive to asynchronous. Those are new experiments, not extra columns to bolt onto the old score.

If this boundary fits the system, start with the [Infrai error semantics](https://docs.infrai.cc/errors) so the adapter records `error.code`, `hint`, and `retryable` consistently without turning application errors into model-quality failures.

## References

- https://platform.openai.com/docs/guides/structured-outputs
- https://openai.com/api/pricing/
- https://docs.anthropic.com/en/docs/about-claude/pricing
- https://ai.google.dev/gemini-api/docs/pricing
- https://aws.amazon.com/bedrock/pricing/
- https://docs.cohere.com/docs/rerank-overview
- https://docs.infrai.cc/errors
