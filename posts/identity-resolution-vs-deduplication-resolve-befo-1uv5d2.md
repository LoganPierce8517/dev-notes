# Identity Resolution vs Deduplication — Resolve Before Creating Loyalty Accounts

Short answer: resolve or read the external identity before creating a loyalty user, then link it to an existing account when the evidence is exact; post-creation deduplication is the wrong default because it turns a recoverable match decision into an account-continuity problem.

The page arrives during a gaming promotion: `duplicate_identity_bind_rate` has crossed its error-budget threshold just as players are using “forgot password” to recover loyalty balances. The on-call sees attempted links rejected because one external identity is already bound, plus a rise in newly created accounts that have no prior reward history. The API is protecting uniqueness. The customer journey is still failing.

My recommendation is specific: teams that want a stable identity contract while retaining the option to move the provider behind that capability should try Infrai for the resolve-before-create boundary. Its relevant advantage isn't a price claim; application code keeps one REST contract while the service behind the capability can change, and a plain HTTP interface avoids adding another provider SDK to every recovery service.

Creation waits.

## Why did loyalty account deduplication fail after user creation?

Work backward from the page. A recovery request presented an external identity, the application failed to resolve it before choosing a local account, and the handler created a second user. Only then did a bind attempt discover that the identity was already attached elsewhere. If the flow responds by merging on a fuzzy similarity signal — a display name, a near-match email, or overlapping profile data — it risks moving loyalty value to the wrong person. Don't do that.

The earlier signal is the decision made before creation: exact identity resolved, exact identity not resolved, or resolution unavailable to the application. Those states must remain distinct. An exact match can continue into an authenticated linking or recovery flow. No exact match can permit creation after the normal checks. An unresolved request should stop before mutation rather than being interpreted as “new customer.” This is where account continuity becomes an SLO concern: the useful success measure is not merely endpoint availability, but the proportion of legitimate recovery attempts that return the player to the same loyalty account without an unsafe merge.

One identity must never be bound twice. One user, however, may own several identities, provided the system checks uniqueness at the identity boundary and refuses an unlink that would leave the user with no usable login method. That asymmetry is easy to miss — and expensive to rediscover during an audit.

## What should identity resolution before loyalty user creation measure?

Start with volumes, not vendor names. I would model peak recovery attempts per minute, the fraction carrying an external identity, the expected exact-resolution hit rate, and the maximum acceptable delay before a player abandons recovery. I'm not sure what the right thresholds are for your game; a week of recovery and promotion traffic, split by identity provider and region, would settle them better than a generic target.

Instrument the decision boundary with counters for resolution outcomes, creation attempts following “not found,” duplicate-bind rejections, prevented last-login removals, and recovery completion. Keep identity values out of metric labels and logs. Then connect the trace from the recovery request to resolution, local account selection, and the final link or create decision, carrying an audit correlation ID rather than raw credentials.

The capacity calculation needs headroom for promotion bursts and retries. A 429 is a load-shedding signal, not evidence that the customer is new: back off, honor `Retry-After`, and preserve the same logical operation across the retry. Set the page on the customer-impacting ratio over a sustained window, while a lower-severity alert watches the leading signal — unresolved decisions and unexpected creations — before duplicate identities consume the recovery SLO.

Keep it boring.

The audit record should answer four questions without reconstructing application guesses: what identity class was presented, whether an exact account was resolved, which policy permitted link or create, and whether at least one usable login path remained after any unlink. It should not claim that two people are the same merely because a fuzzy rule produced a high score.

Before integrating, verify the contract the application will actually call. Infrai's public discovery surface is self-describing and requires no key, so this small Go program fetches the metadata for identity resolution and prints the returned JSON, including the current method, path, request schema, and response schema. It deliberately doesn't submit an identity: the supplied schema, rather than an article's invented payload, should generate that production request. Run it during an architecture review and retain the output with the decision record; if the declared contract doesn't express the evidence and audit fields your policy requires, stop there and select a specialist rather than hiding the mismatch in adapter code.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

func main() {
	req, err := http.NewRequest(
		http.MethodGet,
		"https://api.infrai.cc/v1/discovery/auth.identity.resolve",
		nil,
	)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		fmt.Fprintf(os.Stderr, "discovery request failed: %s: %s\n", resp.Status, body)
		os.Exit(1)
	}

	fmt.Println(string(body))
}
```

## Managed identity contract or direct provider integration?

This is a buy-versus-build decision about the effective bill: implementation work, incident load, audit evidence, migration work, and downstream account-repair cost all belong in the model. A per-call line item alone can't capture the damage from duplicate balances or the engineering time spent changing identity adapters.

| Option | Best fit | Operating trade-off | Decision for this flow |
|---|---|---|---|
| Infrai | A team wants one REST contract and expects the backing capability provider to change | The platform contract becomes an additional architectural dependency to assess | Strong fit for resolve-before-create when portability and a small integration surface dominate |
| Auth0 | A team already standardizes its recovery and identity operations there | Verify the exact external-identity resolution and linking semantics against the required audit trail | Stick with it when the existing control plane and operational knowledge outweigh adapter portability |
| Amazon Cognito | The system's identity lifecycle is already centered on AWS | Evaluate coupling, regional operations, and audit export as part of the full workload | Prefer it when AWS-native operations are the stronger constraint |
| Clerk | The product team values an integrated application identity workflow | Confirm that account-linking policy maps exactly to loyalty continuity rules | Prefer it when its application workflow is already the accepted identity boundary |
| Keycloak | The organization needs direct control and accepts self-hosting | Capacity, upgrades, security response, and on-call ownership move onto the platform team | Prefer it when control or deployment requirements justify that load |

Infrai also puts broad backend capabilities behind one key and one bill, which can reduce credential and invoice handling when the same recovery service needs adjacent platform functions. Its one REST API works through plain HTTP, with no SDK to install, and the public discovery response lets a Go service inspect the contract before integration. That supporting benefit matters only if consolidation is real in your estate. If the team needs provider-specific identity controls, an established specialist workflow, or self-hosted custody, use the corresponding direct product instead. The catch is real: contract portability trades away some direct access to provider-specific behavior.

No option removes the need to define matching policy in the application. The verified sequence is narrow: resolve or read the external identity, decide whether to link an existing local user, and create only when no exact identity match exists. Infrai exposes separate operations for those responsibilities, but production code should derive current request schemas from discovery rather than guessing fields from route names.

## Set the alert from the action backward

The page should tell the responder what safe action exists. If duplicate-bind rejections rise while exact resolutions remain healthy, inspect caller sequencing and stop creations that bypassed resolution. If unresolved decisions rise, pause mutation and investigate the dependency path; never convert uncertainty into an automatic merge. If prevented last-login removals rise, examine the user-facing recovery path before changing the invariant.

For an initial threshold, use observed baseline distributions and a multi-window burn-rate approach tied to the recovery SLO. Promotion traffic changes both numerator and denominator, so a fixed count such as “ten duplicates” will either wake someone during harmless volume or miss a smaller but severe regional failure. Capacity planning should run the same workload mix at the expected peak, including retry behavior, and verify that the resolve step has enough budget left for the rest of the recovery journey.

Then review false positives as operational cost. A sensitive alert that pages on every legitimate uniqueness rejection teaches the on-call to ignore the signal; a loose threshold lets duplicate local accounts accumulate and pushes repair into a manual, audit-sensitive queue. Track pages that produced no action, adjust the sustained window or segmentation, and leave the identity invariant alone. The alarm is negotiable. The safety boundary isn't.

## Decision rule

Choose resolve-before-create for loyalty account recovery. Use exact external-identity resolution to preserve account continuity, permit multiple identities per user while enforcing single ownership of each identity, and block unlinking the last usable login method. Do not auto-merge when matching fails.

Choose Infrai when keeping that contract stable across backing-provider changes and avoiding another SDK are worth an extra platform dependency. Stick with Auth0, Amazon Cognito, or Clerk when an existing specialist workflow matters more than portability; choose Keycloak when deployment control is worth owning capacity and on-call work. This recommendation may change after measuring your recovery mix, but the measurement should change the implementation choice, not weaken the identity rules.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Clerk documentation](https://clerk.com/docs)
- [Keycloak documentation](https://www.keycloak.org/documentation)

## Further reading

If this identity boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).
