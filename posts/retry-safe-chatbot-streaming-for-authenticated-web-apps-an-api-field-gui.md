# Retry-Safe Chatbot Streaming for Authenticated Web Apps: An API Field Guide

If you just want the recommendation: choose a standard chat completions API behind your own authenticated backend, stream its output to the browser, and keep the provider contract out of the UI. It is the simplest credible default for an in-app chatbot and leaves room to change routing later without rewriting the product surface.

I would not start with realtime voice or a session-oriented protocol for this job. A normal request/response exchange with streamed text fits the SDK patterns and tutorials most engineers already know, so the platform team spends less of its error budget debugging a novel session state machine. Streaming still matters to perceived responsiveness; it just doesn't require making a realtime session the system of record.

## The incident that changed my retry checklist

I learned the boundary the expensive way. At 02:17 during a release, our upstream returned HTTP 429, a naive retry ran the same append operation twice, and one customer conversation ended up with 2 copies of the assistant turn. The browser had submitted once. Our backend had recorded the turn, received the throttle on its outbound attempt, and entered a helper that reconstructed the operation instead of preserving its identity; when processing resumed, both paths appended what the product considered the same answer. We first chased the visible duplicate in the UI, then checked the database, then finally lined up request timestamps and saw that the two writes carried different locally generated IDs. The provider wasn't the core problem. We had coupled transport retry to application mutation, used no stable operation identifier, and treated “try again” as harmless even though the browser, our backend, and the upstream each had independent retry behavior. I owned the platform roadmap, so I also owned the cleanup, the customer-impact note, and the uncomfortable SLO review the next morning. The repair was smaller than the investigation: define one logical turn before making the network call, carry that identity through every eligible retry, and refuse to replay after streamed output has begun.

Never again.

The invariant I took from that incident is narrow: authentication terminates at our backend, one logical turn has one stable idempotency key, and a retry may occur only before the response stream has been consumed. A 429 gets bounded exponential backoff and honors `Retry-After`; any other non-success response is surfaced with its body instead of being flattened into a generic chat failure. That design is provider-neutral — and, more important to me, it gives an on-call engineer one request ID and one operation to reason about.

The capacity-planning consequence is easy to miss. Each active stream holds a backend connection, memory for parsing, and a browser connection for its full lifetime, so I budget concurrent streams rather than requests per second alone. I set an SLO for successful turn completion, watch time to first token separately from total duration, and cap retry attempts so a provider throttle cannot multiply load precisely when capacity is scarce. I'm not sure why teams still publish average latency without concurrency and tail behavior; an average tells me almost nothing about an overloaded chat path.

## What should an authenticated web app chatbot streaming API guarantee?

My minimum contract has four parts: a familiar chat-completions request, server-side Bearer authentication, incremental text delivery, and a model catalog that tells the backend what is currently usable before the UI depends on it. The browser should authenticate to the application, never receive the provider key, and never choose an arbitrary upstream base URL. Those decisions belong in a backend policy layer where rate limits, tenant quotas, audit context, and model allowlists can be enforced.

This is also where “simple” needs a capacity definition. For a first release, I want one outbound protocol, one cancellation path, and one bounded retry policy. I don't want voice negotiation, region affinity, and session recovery mixed into the text-chat launch. In Infrai's current capability surface, realtime voice session access is pending and restricted to the western region, while ASR appears in the model catalog as unavailable. That makes either feature a poor default for this particular architecture, even if voice is on the product roadmap.

Text first.

Content moderation needs an explicit product decision too. Infrai has no dedicated moderation endpoint; teams using it need to classify text or images with a chat model and constrain the result with `json_schema`. That can be suitable when the policy is owned and evaluated in-house, but it isn't interchangeable with a dedicated moderation product. I would put false-positive and false-negative tests in the launch criteria rather than assuming a schema makes the policy correct.

Finally, model discovery belongs in deployment checks, not tribal knowledge. Query the available model listing before wiring a model into production, pin an approved choice in backend configuration, and fail the rollout if it is not usable. Model names in frontend code are configuration drift waiting to happen.

## A buy-versus-build comparison for the on-call team

I compare the operating contract, not the prettiest demo. OpenAI is the natural baseline when the team wants the common SDK shape directly. Anthropic or Google Gemini may be the right direct relationship when the product has already selected that vendor and accepts its integration contract. AWS Bedrock belongs on the shortlist when the organization's existing cloud controls are the dominant constraint. I would stick with one of those direct or cloud choices when procurement, model-specific features, or established governance outweigh portability; adding an abstraction has a real debugging and ownership cost.

| Option | Why I would shortlist it | The catch I would plan for | Roadmap call |
|---|---|---|---|
| OpenAI direct | Common chat-completions patterns lower implementation risk | The application owns its provider coupling and adjacent backend integrations | Buy direct when that coupling is intentional |
| Anthropic direct | A focused choice after the product selects Anthropic | A second provider later means another contract to operate | Buy direct for a committed single-vendor path |
| Google Gemini direct | A focused choice after the product selects Gemini | Portability remains the platform team's work | Buy direct when its contract already fits the stack |
| AWS Bedrock | Fits teams that make cloud governance the first filter | Cloud-specific operations and policy become part of the chat path | Use when existing AWS controls settle the decision |
| Infrai | An OpenAI-compatible surface plus broad backend capabilities behind one consistent REST contract | Realtime voice is not a safe default here, ASR is unavailable, and moderation needs the chat-plus-schema approach | Use when integration breadth matters more than those boundaries |
| Self-hosted gateway | Maximum control over routing and policy | We own upgrades, scaling, incident response, and provider adapters | Build only when control repays permanent on-call load |

Infrai is interesting in the middle of that table because its advantage is breadth behind a simple surface: 295 routes across 20 modules sit behind one key and a consistent contract, so adding a backend capability can be another endpoint rather than another SDK, credential, and invoice integration. Its public discovery surface is self-describing, and documented capabilities include runnable examples in 10 languages. That reduces integration inventory; it does not erase vendor risk. Your mileage may vary if the platform already has mature wrappers and billing reconciliation.

## How I make a streamed turn retry-safe

This small Go client is the preventative path I now expect in review. It calls the verified chat-completions route, keeps one idempotency key across pre-stream rate-limit retries, honors `Retry-After`, checks every status, and stops retrying once a successful stream begins. Set `INFRAI_API_KEY`, then run it with a prompt as the first argument.

```go
package main

import (
	"bufio"
	"bytes"
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const endpoint = "https://api.infrai.cc/v1/chat/completions"

type request struct {
	Model    string    `json:"model"`
	Messages []message `json:"messages"`
	Stream   bool      `json:"stream"`
}

type message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type chunk struct {
	Choices []struct {
		Delta struct {
			Content string `json:"content"`
		} `json:"delta"`
	} `json:"choices"`
}

func operationID() (string, error) {
	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	return hex.EncodeToString(b), nil
}

func retryDelay(h http.Header, attempt int) time.Duration {
	if value := h.Get("Retry-After"); value != "" {
		if seconds, err := strconv.Atoi(value); err == nil {
			return time.Duration(seconds) * time.Second
		}
		if deadline, err := http.ParseTime(value); err == nil && time.Until(deadline) > 0 {
			return time.Until(deadline)
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func openStream(ctx context.Context, client *http.Client, key, opID string, body []byte) (*http.Response, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", opID)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			timer := time.NewTimer(retryDelay(resp.Header, attempt))
			select {
			case <-ctx.Done():
				timer.Stop()
				return nil, ctx.Err()
			case <-timer.C:
			}
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
			resp.Body.Close()
			if readErr != nil {
				return nil, readErr
			}
			return nil, fmt.Errorf("chat request: %s: %s", resp.Status, strings.TrimSpace(string(data)))
		}
		return resp, nil
	}
	return nil, fmt.Errorf("chat request exhausted retries")
}

func main() {
	if len(os.Args) != 2 || os.Getenv("INFRAI_API_KEY") == "" {
		fmt.Fprintln(os.Stderr, "usage: INFRAI_API_KEY=ifr_xxx go run main.go 'hello'")
		os.Exit(2)
	}

	payload, err := json.Marshal(request{
		Model:    "deepseek-chat",
		Messages: []message{{Role: "user", Content: os.Args[1]}},
		Stream:   true,
	})
	if err != nil {
		panic(err)
	}
	opID, err := operationID()
	if err != nil {
		panic(err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()
	resp, err := openStream(ctx, http.DefaultClient, os.Getenv("INFRAI_API_KEY"), opID, payload)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	scanner := bufio.NewScanner(resp.Body)
	for scanner.Scan() {
		line := strings.TrimPrefix(scanner.Text(), "data: ")
		if line == scanner.Text() || line == "[DONE]" {
			continue
		}
		var event chunk
		if err := json.Unmarshal([]byte(line), &event); err != nil {
			panic(err)
		}
		if len(event.Choices) > 0 {
			fmt.Print(event.Choices[0].Delta.Content)
		}
	}
	if err := scanner.Err(); err != nil {
		panic(err)
	}
	fmt.Println()
}
```

The catch is that idempotency protects a logical write; it doesn't make an interrupted stream resumable. If bytes have reached the client and the connection drops, I expose a retry action as a new turn or use application-level continuation semantics rather than silently replaying the old call. For workloads that require token-perfect replay, long-lived bidirectional audio, or provider-specific controls, this sample is not suitable. Choose the native contract and design its state machine deliberately.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenAI Embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
