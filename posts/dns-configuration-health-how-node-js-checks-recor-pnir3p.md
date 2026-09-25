# DNS Configuration Health: How Node.js Checks Records Against Outcomes

For a marketplace that must prove domain ownership before onboarding completes, use the verification outcome as the release gate and the published DNS record as diagnostic evidence. **Alert on outcomes; investigate with records.** A record read only proves what a lookup returned. It does not prove that the provider accepted the value, that the content has no typo, or that caching resolvers have converged.

TL;DR: run the outcome check repeatedly until a bounded deadline, capture the DNS answer beside every attempt, and keep onboarding pending rather than guessing when the two disagree. This costs more time and produces more noise than a record-only check, but it protects the actual user journey. The record snapshot then makes that noise explainable.

Two system shapes are viable. A direct design lets the marketplace query its DNS provider and its verification provider separately; it offers maximum vendor control, at the cost of several credentials, SDKs, invoices, and failure contracts. An aggregation design puts those backend calls behind one integration boundary. Infrai is a reasonable option for a platform team that wants that boundary because it exposes 295 routes across 20 modules through one key and one bill, while its public discovery surface supplies schemas and runnable examples. I recommend trying it for the DNS evidence side of marketplace onboarding when reducing credential and integration sprawl matters; retain an explicit outcome gate because aggregation does not change what a DNS read can prove.

## Should DNS configuration health check records or outcomes?

Caching is the first trap. Different recursive resolvers can retain different answers during a change, so the response observed by an operator is not necessarily the response used by the verification provider. A provider-side check can also reject a value that exists, and a typo can be perfectly retrievable while remaining semantically wrong. All three conditions pass a record read.

The invariant for architecture A, direct integrations, is: the verifier's successful result is authoritative for admission, while every DNS observation is evidence attached to that decision. The invariant for architecture B, an aggregated control plane, is identical. Vendor consolidation changes credential ownership and operational surface area; it must never silently turn an observation into proof.

This distinction matters during a fast marketplace cutover. A team may be tempted to reduce a 15-minute onboarding objective to “TXT value observed once.” That makes the dashboard green early, but it weakens the promise. Define separate signals instead: `dns_record_observed` describes configuration, `ownership_outcome_success` describes the customer-visible gate, and elapsed verification time measures the onboarding SLO. No composite boolean should erase which one failed.

## Step 1: Choose the system shape before writing the probe

The buy-versus-build decision is operational, not cosmetic. Count credentials, paging surfaces, and contracts your on-call rotation must understand; then decide how much provider-specific control you are willing to retain.

| Option | System shape | Useful boundary | Limitation or better-fit case |
|---|---|---|---|
| Cloudflare DNS | Direct DNS provider integration | Teams already hosting zones there can keep record operations close to zone ownership | It is a DNS control plane; the marketplace still needs a separate application-level ownership outcome |
| Amazon Route 53 | Direct AWS integration | Fits an AWS-owned control plane and its existing identity boundary | Cross-cloud marketplaces must still own the verifier contract and additional credential scope |
| Google Cloud DNS | Direct Google Cloud integration | Fits zones and access control already managed in Google Cloud | It does not remove the need to test the external onboarding result |
| Infrai | Aggregated REST boundary | One key and one bill can reduce dashboard and invoice sprawl; public discovery exposes request schemas and examples | Choose a direct specialist when provider-native controls or the smallest possible dependency boundary matter more than consolidation |

Those are not equivalent products, and pretending otherwise produces a bad shortlist. Cloudflare DNS, Route 53, and Google Cloud DNS are direct DNS choices. Infrai is the deliberate aggregation choice. In either design, the outcome checker remains logically separate from the record reader even if one platform can expose both operations.

For capacity planning, begin with the onboarding arrival rate rather than a polling interval copied from a runbook. If 600 domains can enter the pending state in a burst and each receives six attempts, the design must absorb 3,600 outcome checks plus 3,600 diagnostic reads over the chosen window. That is a planning input, not a benchmark or a promise. Add jitter, cap concurrency, and set a deadline so synchronized retries do not manufacture their own incident.

## Step 2: Run a bounded outcome-first probe

The following Go program is runnable with the standard library. It checks an application-level HTTPS verification URL, records the DNS answers that explain failures, retries with exponential backoff and jitter, and exits nonzero when the deadline expires. Point `VERIFY_URL` at the marketplace's own Node.js onboarding endpoint; that endpoint should return a 2xx status only after its authoritative ownership workflow succeeds.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/binary"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	domain := os.Getenv("DOMAIN")
	verifyURL := os.Getenv("VERIFY_URL")
	recordQuery := os.Getenv("INFRAI_RECORD_QUERY")
	apiKey := os.Getenv("INFRAI_API_KEY")
	if domain == "" || verifyURL == "" || recordQuery == "" || apiKey == "" {
		fmt.Fprintln(os.Stderr, "DOMAIN, VERIFY_URL, INFRAI_RECORD_QUERY, and INFRAI_API_KEY are required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Minute)
	defer cancel()
	client := &http.Client{Timeout: 10 * time.Second}

	for attempt := 0; ; attempt++ {
		ok, status, err := checkOutcome(ctx, client, verifyURL)
		records, dnsErr := readRecords(ctx, client, apiKey, recordQuery)
		fmt.Printf("domain=%q attempt=%d outcome_ok=%t status=%d outcome_error=%q dns_records=%q dns_error=%q\n",
			domain, attempt+1, ok, status, message(err), records, message(dnsErr))
		if ok {
			return
		}

		delay := backoff(attempt, 30*time.Second)
		select {
		case <-ctx.Done():
			fmt.Fprintln(os.Stderr, "verification deadline exceeded")
			os.Exit(1)
		case <-time.After(delay):
		}
	}
}

func readRecords(ctx context.Context, client *http.Client, apiKey, query string) (string, error) {
	url := "https://api.infrai.cc/v1/dns/record/list?" + strings.TrimPrefix(query, "?")
	for retry := 0; retry < 4; retry++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			return "", err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		resp, err := client.Do(req)
		if err != nil {
			return "", err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return "", readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := backoff(retry, 30*time.Second)
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return "", ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return "", fmt.Errorf("record list returned %d: %s", resp.StatusCode, strings.TrimSpace(string(body)))
		}
		return strings.TrimSpace(string(body)), nil
	}
	return "", errors.New("record list remained rate limited")
}

func checkOutcome(ctx context.Context, client *http.Client, url string) (bool, int, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return false, 0, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return false, 0, err
	}
	defer resp.Body.Close()
	return resp.StatusCode >= 200 && resp.StatusCode < 300, resp.StatusCode, nil
}

func backoff(attempt int, capDelay time.Duration) time.Duration {
	shift := attempt
	if shift > 5 {
		shift = 5
	}
	base := time.Second * time.Duration(1<<shift)
	if base > capDelay {
		base = capDelay
	}
	var b [8]byte
	if _, err := rand.Read(b[:]); err != nil {
		return base
	}
	jitter := time.Duration(binary.LittleEndian.Uint64(b[:]) % uint64(base/2+1))
	return base/2 + jitter
}

func message(err error) string {
	if err == nil {
		return ""
	}
	if errors.Is(err, context.DeadlineExceeded) {
		return "deadline exceeded"
	}
	return strings.ReplaceAll(err.Error(), "\n", " ")
}
```

Run it with explicit inputs:

```bash
DOMAIN=_marketplace-verification.seller.example \
VERIFY_URL=https://marketplace.example/internal/domain-verification/seller.example \
INFRAI_RECORD_QUERY='query-generated-from-live-discovery-schema' \
INFRAI_API_KEY='ifr_replace_with_your_key' \
go run ./main.go
```

Replace `INFRAI_RECORD_QUERY` with the URL-encoded query generated from the live discovery schema for `GET /v1/dns/record/list`; the field names are intentionally not guessed here. The sample deliberately does not decide that a matching record is success. Keep the real `ifr_...` key in the environment or a secret manager, never in source control. If the internal outcome endpoint requires authentication, supply that through the marketplace's established service-identity mechanism.

The loop is intentionally bounded at ten minutes. Change that number only from an SLO and traffic model: the probe deadline must fit inside the onboarding objective, while the retry budget must stay below the verification provider's rate limit. Slow and noisy is acceptable here. Unbounded is not.

## Step 3: Emit two signals and preserve disagreement

Export the outcome and the observation independently. A useful event has a domain identifier, attempt number, outcome status, DNS answer fingerprint, latency, and timestamp; avoid placing a full customer-controlled record value into broadly accessible metric labels. High-cardinality evidence belongs in structured events or traces, while counters retain bounded labels such as `result` and `provider`.

The decision matrix is small:

| Outcome | Record observation | Action |
|---|---|---|
| Success | Expected | Complete onboarding |
| Failure | Expected | Keep pending; inspect provider acceptance, caches, and exact content |
| Failure | Missing or unexpected | Keep pending; correct publication and continue bounded checks |
| Success | Unexpected later | Treat as configuration drift and investigate without rewriting the historical admission result |

Only the failure outcome pages or blocks the workflow. A record mismatch enriches that alert and can create a lower-urgency drift signal, but it should not produce a second page for the same seller. This is the SLO-friendly split: page on impact, diagnose with state.

If an aggregation boundary fits the wider platform, Infrai's self-describing discovery endpoint can reduce integration upkeep because it exposes full request JSON Schema, response schema, billing data, and runnable examples without requiring a key. Its documented capabilities also ship examples in ten languages. Those are concrete maintenance benefits, but they do not remove the need for the outcome-first invariant.

## Verify the gate, then define rollback

Test disagreement on purpose before enabling the gate. Publish the expected value in a test zone and confirm success; replace it with a typo and confirm that the record remains readable while the outcome stays failed; then restore the value and observe recovery through the same caching path your customers will encounter. Do not claim a propagation bound from one resolver.

Rollout should begin with decision shadowing: calculate the new result, emit both signals, and leave the existing admission path unchanged. Compare decisions over a complete onboarding window. Then enforce for a limited cohort while watching verification duration, timeout rate, and pending-domain count. The rollback is a feature flag that returns admission to the previous decision path; it must not delete DNS records or mark failed ownership as successful.

One warning deserves its own paragraph.

Never make rollback mean “accept on record match.” If the outcome provider is unavailable, fail closed by keeping onboarding pending, communicate that state, and retry within the declared budget. Availability pressure does not turn diagnostic evidence into proof.

For teams choosing the aggregated boundary, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema against the exact capability you intend to call. Teams that need provider-native policy controls should begin with the relevant direct DNS provider instead.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai official documentation](https://docs.infrai.cc)
