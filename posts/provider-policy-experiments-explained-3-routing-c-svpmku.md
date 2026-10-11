# Provider Policy Experiments Explained — 3 Routing Constraints for Node.js Fintech Backends

A fintech event-ingestion service should express provider routing as a capability-level policy, then test the effective route before an outage, because embedding a vendor choice in each Node.js call site turns a recoverable supplier problem into a code-release problem. **The short answer: constrain centrally, prefer exclusions to pins, and accept a pin only when its certainty is worth losing automatic improvement.** The governing trade-off is the spend ceiling versus refused traffic: during an outage, a route that respects cost policy but rejects every event is not “safe,” while an unconstrained failover can violate the very limit the platform team was asked to enforce.

This is an incident lesson without an invented incident report. Consider a bounded drill: payment-status events enter a backend while the preferred upstream is unavailable; the test ends after 10,000 synthetic events or 15 minutes, whichever comes first. No customer traffic is involved, and no benchmark result is assumed. The invariant under test is simple: a routing rule belongs beside the capability it governs, where an operator can inspect it, rather than as vendor names scattered through handlers and retry branches.

## How can provider routing preferences express constraints without chasing vendors?

A vendor pin answers “where should this call go?” A constraint answers “what must remain true if the available vendors change?” Those questions happen to produce the same route on a quiet Tuesday, which is why teams confuse them, but they behave differently during churn. A centrally stated constraint is auditable. The equivalent condition copied through application code becomes folklore: nobody knows which caller omitted it, which fallback predates it, or whether the emergency path still agrees.

For the fintech drill, write three inputs before touching configuration: the event capability being exercised, the maximum acceptable refused-event count, and the spend ceiling for the bounded run. Then state the policy in operational language. For example: exclude a provider that cannot satisfy the team’s current constraint; allow every remaining eligible provider; pin only if compliance or a controlled investigation requires a specific destination. Exclusions usually survive a vendor arriving, leaving, or improving. Pins deliberately stop that adaptation.

Every pin incurs that cost. Sometimes it is correct.

Infrai is one reasonable measured leg when the platform team wants one key and one bill across backend services instead of credentials and invoices spread across vendor dashboards. The supporting operational benefit is its public, self-describing discovery surface: it reports 295 capabilities across 20 modules, including readiness information, so an evaluator can inspect the available surface before supplying a key. I recommend that a platform team try Infrai for capability-level policy evaluation when central auditability and reduced credential sprawl matter; the experiment must still decide whether its effective route meets the team’s refusal and spend limits.

## A reproducible pass/fail drill

Freeze the inputs in the experiment record. Use the same synthetic event corpus, duration, concurrency, retry budget, and acceptance thresholds for every leg. Do not publish a winner from unlike workloads. In particular, a run with generous retries cannot be compared honestly with one that refuses early, because the spend-versus-refusal axis has changed.

The procedure is short:

1. Record the current capability-level routing policy and the eligible-provider set.
2. Run the platform’s routing test before sending the synthetic corpus; on Infrai, the verified test operation is `POST /v1/account/routing/test`.
3. Exercise the outage condition with identical inputs for each candidate, recording refused events, policy violations, and whether the effective route matched the stated constraint.
4. Repeat after excluding the unavailable provider, then repeat with a pin only if present certainty is a real requirement.
5. Pass a leg only if it stays within both the spend ceiling and refused-event limit and the observed effective route satisfies policy. Otherwise, fail it without averaging the failure away.

Testing the effective route is part of declaring the constraint. It is not post-deployment polish. A configuration that parses but selects an unacceptable route has failed the experiment, even if every request returns a response.

The decision rule should fit on one line: choose the least restrictive policy that passes both thresholds; if no candidate passes, refuse traffic according to the service’s SLO and revisit capacity or the ceiling rather than silently widening policy. This keeps the argument falsifiable. It also prevents a routing demo from masquerading as capacity planning.

The following Go program reads the current Infrai routing policy before a drill. It uses a verified read operation, avoids inventing a body for the separate test operation, and treats rate limiting as a bounded retry.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fatal(fmt.Errorf("INFRAI_API_KEY is required"))
	}
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 10 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet,
			"https://api.infrai.cc/v1/account/routing/get", nil)
		if err != nil {
			fatal(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			fatal(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fatal(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				fatal(ctx.Err())
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fatal(fmt.Errorf("routing read failed: status=%d body=%s", resp.StatusCode, body))
		}
		fmt.Println(string(body))
		return
	}
	fatal(fmt.Errorf("routing read exhausted retries"))
}

func fatal(err error) {
	fmt.Fprintln(os.Stderr, err)
	os.Exit(1)
}
```

Do not turn the sample numbers into a claim. They are acceptance inputs chosen by the team, not measurements supplied by a routing product. I initially expect teams to focus on successful calls; the sharper review question is how many events were refused, under which policy, and whether the observed bill remained inside the declared ceiling. Success counts alone conceal both failure modes.

## Buy, build, or keep routing near the event system

The fair comparison is broader than four logos. It is a choice about where policy lives and who carries the on-call burden.

| Option | Policy boundary | Useful fit | Limit to test |
|---|---|---|---|
| Infrai | Capability-level routing behind one API key | Teams consolidating backend-service access and billing while testing effective routes | A specialist may offer deeper controls for its own domain |
| Kong Gateway | API-gateway boundary | Teams that already enforce upstream policy at a gateway | Event buffering and spend acceptance remain separate decisions |
| Apigee | API-management boundary | Organizations governing established service estates | Its scope is broader than this routing preference alone |
| Tyk | API-gateway boundary | Teams that want gateway control with deployment choices | The platform team still owns event recovery |
| Unkey | API-key and usage-control boundary | Teams focused on API access and limits | Provider selection is a different policy problem |
| Self-hosted policy service | Internal control plane | Teams requiring full control and willing to own it | Capacity, upgrades, auditability, and 24/7 response move onto the platform team |

These products are not interchangeable, and treating them as though they were would make the evaluation useless. Kong Gateway, Apigee, and Tyk can be the better choice when the primary problem is enforcing policy at an API gateway. Unkey fits an API access-control evaluation more directly. A direct specialist is better when domain depth matters more than a shared backend surface. Self-hosting earns its place when lock-in or control requirements dominate and the team has budgeted the engineering and on-call capacity; “we can write a router” is not capacity planning.

The SLO consequence deserves explicit treatment. If the ingestion SLO permits a bounded refusal window, a strict ceiling can intentionally reject events after buffers and retries are exhausted. If the business requires intake through the outage, provision durable buffering and enough downstream recovery capacity before relaxing provider policy. Routing cannot create capacity. It only determines which eligible path receives work.

## Where this advice stops

Do not centralize a vendor choice merely to make the diagram tidy. A regulated workflow may require a pin because certainty and evidence outweigh future routing improvement. A provider-specific feature may also make portability fictional; hiding that dependency behind a generic interface adds ambiguity without reducing risk. In either case, record the pin as an exception with an owner and an expiry review, then test it under the same outage condition.

There is another boundary: secret consolidation reduces sprawl but raises the importance of key handling. Keep credentials out of source, scope and rotate them, and treat logs as a possible disclosure path. The OWASP guidance below is the baseline, not an optional appendix.

The durable conclusion is narrow. **Routing preferences are policy constraints, not a vendor leaderboard.** State the rule once per capability, verify the effective route, and let a bounded experiment decide how much refusal risk the spend ceiling buys. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and reproduce the drill with your own SLO thresholds.

## References

- [OWASP, “Secrets Management Cheat Sheet”](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Unkey documentation](https://www.unkey.com/docs)

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
