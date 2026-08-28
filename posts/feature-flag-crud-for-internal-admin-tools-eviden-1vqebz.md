# Feature Flag CRUD for Internal Admin Tools — Evidence Before Convenience

Short answer: for a simple Node.js and Express internal tool, put feature flag CRUD in an admin dashboard backed by an append-only change record, optimistic version checks, and a reversible disable operation; the four buttons are easy, but preserving enough trustworthy evidence to reconstruct an e-commerce incident is the real requirement.

A simple Node.js and Express internal tool can expose set, list, delete, and toggle actions without becoming a second control plane. Keep the browser thin. Put authorization, validation, version checks, mutation, and audit emission behind one server-side service boundary, then make every dashboard action call that boundary. This is less convenient than letting each route update a row directly — deliberately so — because a flag change that affects checkout needs a durable answer to who changed what, why, and from which prior value.

The deciding SLO question is not “did the button respond?” It is “after a customer reports a failed order, can an operator distinguish a flag transition from an application defect without guessing?”

## What the dashboard must preserve

Treat the current flag value as a projection, not the complete record. A useful record for an internal e-commerce tool has a stable key, an enabled state, an integer version, an update time, and an actor identifier. Each accepted mutation should also append an event containing the operation, the value before the change, the value after it, the submitted reason, and the resulting version. A list view reads the projection; incident reconstruction reads the events.

That split controls noise. Logging every evaluation of a frequently read flag would bury the rare administrative changes that matter, while recording only the latest value would erase the sequence an investigator needs. The dashboard should therefore observe mutations at high fidelity and leave request-level flag evaluation to the application telemetry already attached to the checkout flow. Correlation belongs at the edge: include the active flag version in the order or request diagnostic context when that evidence is useful, but don't copy customer details into the flag key or free-form reason.

Deletion deserves precise language. In the operator view, delete can remove a flag from the active set; in the evidence store, it should append a tombstone event instead of silently erasing the prior transition chain. That choice creates a privacy constraint — event fields should contain operational identifiers rather than customer personal data. GDPR Article 17 establishes a right to erasure under stated conditions and also lists exceptions, so retention and erasure policy need review by the organization responsible for the data; an audit log is not permission to retain personal data indefinitely.

Small distinction, large consequence.

## How should a Node.js Express admin dashboard set, list, delete, and toggle feature flags?

Use four explicit service operations even if Express exposes them through a compact router. `Set` creates a missing flag or assigns a requested state, `List` returns active projections, `Toggle` flips the state only when the submitted version still matches, and `Delete` creates a tombstone under the same version rule. The Express handlers should parse input, obtain the authenticated actor from server-side identity context, call the service, and translate a stale-version result into a conflict response. They shouldn't trust an actor name sent by the browser.

The version check closes a mundane but dangerous race. Imagine two operators viewing checkout flag version 12. One disables it during an incident; the other, working from the stale page, presses toggle a moment later. Without compare-and-set semantics, the second action can reverse the mitigation while both screens report success. With them, only the first mutation advances version 12 to 13, and the second receives a conflict, refreshes, and requires a conscious new decision. No distributed lock is needed in the dashboard layer, but the backing store must make the state update and event append one atomic operation.

The following Go example is intentionally the policy core rather than a complete web server. The task's concrete application may use Node.js and Express, yet keeping transport code out of this example makes the contract obvious: adapters in any runtime must preserve these invariants rather than improvise them per route.

```go
package flags

import (
	"context"
	"errors"
	"strings"
	"time"
)

var (
	ErrInvalid = errors.New("flag key, actor, and reason are required")
	ErrStale   = errors.New("flag version is stale")
)

type Flag struct {
	Key       string
	Enabled   bool
	Version   uint64
	UpdatedAt time.Time
	UpdatedBy string
}

type Change struct {
	Key       string
	Operation string
	Before    *bool
	After     *bool
	Version   uint64
	Actor     string
	Reason    string
	At        time.Time
}

type Store interface {
	ListActive(context.Context) ([]Flag, error)
	SetAndAppend(context.Context, string, bool, *uint64, Change) (Flag, error)
	DeleteAndAppend(context.Context, string, uint64, Change) error
}

type Service struct {
	store Store
	now   func() time.Time
}

func (s Service) List(ctx context.Context) ([]Flag, error) {
	return s.store.ListActive(ctx)
}

func (s Service) Set(
	ctx context.Context,
	key string,
	enabled bool,
	expectedVersion *uint64,
	actor string,
	reason string,
) (Flag, error) {
	if strings.TrimSpace(key) == "" || strings.TrimSpace(actor) == "" || strings.TrimSpace(reason) == "" {
		return Flag{}, ErrInvalid
	}

	change := Change{
		Key: key, Operation: "set", After: &enabled,
		Actor: actor, Reason: reason, At: s.now(),
	}
	return s.store.SetAndAppend(ctx, key, enabled, expectedVersion, change)
}

func (s Service) Toggle(
	ctx context.Context,
	current Flag,
	actor string,
	reason string,
) (Flag, error) {
	next := !current.Enabled
	change := Change{
		Key: current.Key, Operation: "toggle",
		Before: &current.Enabled, After: &next, Version: current.Version + 1,
		Actor: actor, Reason: reason, At: s.now(),
	}
	return s.store.SetAndAppend(ctx, current.Key, next, &current.Version, change)
}

func (s Service) Delete(
	ctx context.Context,
	current Flag,
	actor string,
	reason string,
) error {
	change := Change{
		Key: current.Key, Operation: "delete",
		Before: &current.Enabled, Version: current.Version + 1,
		Actor: actor, Reason: reason, At: s.now(),
	}
	return s.store.DeleteAndAppend(ctx, current.Key, current.Version, change)
}
```

The storage adapter owns the hardest guarantee: `SetAndAppend` and `DeleteAndAppend` must either commit both the projection and its change event or commit neither. A unique constraint on key plus resulting version gives each accepted transition one identity. Return the new version to Express and then to the browser, where it replaces the version attached to the row. Don't implement toggle as a client-side read followed by an unconditional set; that is exactly the stale-page race in a nicer shirt.

There is one deliberate sharp edge in this compact sample: authorization isn't part of `Service` because identity and policy systems vary. It still must execute before any method is called. Separating it here is an interface boundary, not an invitation to omit it.

## Signal quality, access, and retention are one design problem

An admin dashboard is an observability producer. Design its events with the same skepticism applied to any production signal: define the question each field answers, cap unbounded input, reject missing reasons, normalize keys, and never use free text as the only machine-filterable description of an operation. The useful event is compact enough to scan during an incident and specific enough to join with application evidence by flag key and version.

For checkout, a mutation event might say that `checkout.address_validation` changed from enabled to disabled at version 43, by an authenticated operations identity, with a reason tied to an incident identifier. It should not contain a shopper's address, email, payment details, or copied support conversation. This is a proposed event shape, not a claim that one schema fits every retention regime; I'm not sure an organization's existing incident identifier is safe to retain until its data classification and deletion behavior have been checked.

Access should be narrower than visibility. Many engineers may need read access during diagnosis, while a smaller on-call group receives mutation permission; delete can require an additional policy decision because it changes the active configuration and the operator's mental model. Keep those choices server-side. Hiding a button in the browser is useful interface design, but it isn't authorization.

Capacity planning matters even for an “internal” tool. Estimate mutation volume from operators and deployments, event retention from the reconstruction window, and read pressure from both the dashboard and incident queries. The current-state table stays small; event history grows with changes, not evaluations. If a rollout system generates many automated mutations, give machine actors distinct identities and reasons so their traffic doesn't masquerade as human action. Your mileage may vary on the retention window, and the right number comes from the organization's incident lookback needs, legal obligations, and storage budget rather than a generic default.

## Should this remain simple, or become a control plane?

Build the small tool when the flag set is bounded, the authorization model is already available, and the team can own atomic storage, backups, schema changes, and on-call response. It isn't suitable when flags drive large staged rollouts, many services need low-latency evaluation, or policy requires approval workflows the team cannot maintain. In those cases, choose a managed or established self-hosted control plane whose documented behavior matches those requirements; keep the incident-evidence contract at the integration boundary so a future migration doesn't rewrite the investigation workflow.

| Decision | Small internal service | Established control plane |
|---|---|---|
| Primary fit | A bounded administrative surface with a few explicit mutations | Broad distribution, rollout, and governance requirements |
| On-call ownership | Your team owns storage correctness, access policy, backup, and recovery | Responsibility is shared according to the chosen operating model |
| Lock-in pressure | Mostly your event schema and storage adapter | Evaluation semantics, client integration, and exported history need review |
| Evidence test | Prove every accepted mutation has one matching event | Prove exported or retained events preserve actor, reason, before, after, and version |
| Exit criterion | Operational burden or feature demand exceeds the team's SLO budget | Contract, deployment, or migration constraints exceed the value delivered |

The catch is ownership. A few Express handlers can look cheaper on a roadmap while quietly adding a database, access-control surface, backup path, and incident dependency to the platform team's pager. Conversely, adopting a larger control plane for four slow-moving flags can add integration and governance work that produces no better evidence. Price isn't the deciding metric; total on-call work and the ability to meet the reconstruction SLO are.

Use a written exit criterion before either choice. Otherwise “simple internal tool” tends to become a permanent system by accident.

## Verification, deployment, and rollback

Test the invariant below the HTTP layer. Run concurrent mutations from the same starting version and assert that exactly one succeeds, one new current version exists, and one matching change event exists. Test a storage failure between the proposed projection update and event append; the transaction should leave neither visible. Then test authorization denial, invalid keys, blank reasons, deletion of a stale version, list behavior after a tombstone, and event ordering. These cases are more valuable than a screenshot of four working buttons.

Deploy in two stages. First ship read-only list access and event inspection against non-customer test flags, so operators can validate identity display, ordering, and diagnostic usefulness without changing configuration. Then enable mutation permission for a restricted group and run a controlled set-toggle-toggle-delete sequence. Record the expected versions before the exercise and query the evidence afterward. The verification fails if a visible state lacks an event, an event lacks an authenticated actor, or a stale write changes state.

Rollback is a state transition, not a database edit. To reverse an unsafe flag change, submit a new mutation from the latest version with a reason that points to the incident; retain the original event and the reversal. To roll back the dashboard release itself, remove mutation access or route operators back to read-only mode while leaving the evidence store intact. This keeps the incident timeline legible and avoids teaching responders to bypass the system precisely when its evidence matters most.

Be strict here.

Before declaring the tool production-ready, rehearse restoration of both the current projection and the event history, confirm that the restored projection can be derived or checked against the ordered events, and measure the time required against the team's recovery objective. If restoration recovers the button state but loses who changed it and why, the dashboard has recovered configuration while failing its actual job.

## References

- GDPR Article 17, “Right to erasure”: https://gdpr-info.eu/art-17-gdpr/
