# Node.js Receipt Retention: AI-Generated Images, Private Buckets, and Presigned Downloads

Short answer: for a fintech receipt pipeline, keep the generated image in a private bucket, store its immutable object key and retention state in your database, and issue a short-lived presigned download URL only after the caller passes authorization. The important decision is deletion, not the upload call. A URL is a temporary capability; it is not an audit record.

The incident pattern is easy to miss: a receipt flow leaves two artifacts for one job because the database row is gone while a retry has written a second object under a new random key. Nothing is visibly broken to the customer. The storage bill and the audit question are the problem. The invariant is simple: one receipt job gets one planned key, one retention deadline, and one recorded deletion outcome. Retries may repeat an operation, but they must not create a new identity for the same receipt. That is a governance problem before it becomes a storage problem, and the repair belongs in the data model rather than in a later bucket sweep.

## What breaks first when Node.js stores AI-generated images in object storage?

The application should retain the object key, tenant or account identifier, content type, byte count, content digest, creation time, retention deadline, and a deletion state. It should not retain the presigned URL. The URL will expire, while the key and the decision that governs its lifetime need to remain useful to an auditor months later.

For receipts, make the key boring and deterministic. A shape such as `receipts/{account_id}/{receipt_id}/original.png` gives an operator a bounded namespace for reconciliation and cleanup. The receipt ID must already exist before the image worker starts. If the worker times out after an upload and retries, it addresses the same logical object; a random key would turn uncertainty into orphaned data.

There are two separate authorization checks. The API checks whether the requester may view the receipt. Object storage then receives a narrow, expiring GET capability. Never send a bucket credential to a browser, and never treat possession of a database row as proof that the caller may read another account's row. The user-visible retrieval SLO includes authorization, URL creation, and object delivery, so measure the complete path.

Deletion needs an explicit state machine rather than a cron job that quietly removes files:

| State | Meaning | Required evidence |
| --- | --- | --- |
| `active` | The receipt is within its retention period | Key, digest, and deadline |
| `eligible` | Policy permits deletion | Policy version and eligibility time |
| `deleting` | A worker has claimed the deletion attempt | Attempt ID and timestamp |
| `deleted` | The object and database state agree | Completion time and object key |
| `review` | The worker cannot establish the expected result | Error details and an operator decision |

That last state matters. A deletion response observed by the worker is not the same thing as a deletion record that can be reconciled. Keep the attempt ID and avoid silently marking an object deleted when the network result was ambiguous.

## The deletion ledger is the system of record

Treat retention as a policy input, not as an expiry embedded in a download URL. A URL lasting 5 minutes controls exposure for one read; it does not decide whether the original receipt may remain in storage. The database owns that distinction.

Keep it separate.

A worker can claim an `eligible` row, transition it to `deleting`, delete the deterministic key, and then record `deleted`. If the process stops between the object operation and the database update, reconciliation should find the row and retry the same key. That is why the operation must be repeatable and why the key must be stable. At-least-once processing is ordinary; unbounded orphan creation is optional. The longer failure sequence deserves attention: a model worker may finish rendering, the upload may reach storage, the client may time out before receiving the worker result, and a retry may then create another object if the key is generated late. The database cannot reconcile two opaque keys back to one receipt without extra evidence, while a preassigned key makes the ambiguity visible as one pending operation. That is a small design choice with an unusually large effect on deletion review, tenant accounting, and capacity forecasts.

The failure modes worth testing are not exotic:

- the image generator finishes but the upload acknowledgement is lost;
- the deletion worker runs twice for one receipt;
- a receipt is requested after its URL has expired;
- a browser has the right URL but the bucket's CORS policy does not allow the requesting origin;
- a policy change shortens retention while an old worker still has a stale schedule.

The browser case is a contract between the origin, request method, headers, and response headers. MDN's CORS guide explains the browser enforcement model; test the actual staging origin and headers rather than assuming a signed URL bypasses CORS. It does not.

The capacity calculation should be equally plain: active receipt count multiplied by average original bytes, plus the retry and reconciliation allowance, with a separate estimate for any thumbnails. Track object count and bytes by retention cohort. If the bucket grows while `active` and `eligible` rows remain flat, the key invariant or the cleanup ledger deserves investigation.

## Test the uncertain upload before you automate cleanup

The storage client below is intentionally small. It leaves bucket authentication and SDK details behind an interface, while making the important behavior visible: claim a row, delete the known key, and record the result. The application around it can be Node.js; the concurrency rule is language-independent.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

var ErrNotFound = errors.New("object not found")

type Receipt struct {
	ID        string
	ObjectKey string
}

type ReceiptStore interface {
	ClaimEligible(ctx context.Context, now time.Time) (Receipt, error)
	MarkDeleted(ctx context.Context, id string, at time.Time) error
	MarkReview(ctx context.Context, id string, reason string) error
}

type ObjectStore interface {
	Delete(ctx context.Context, bucket, key string) error
}

func deleteOne(ctx context.Context, receipts ReceiptStore, objects ObjectStore, bucket string) error {
	receipt, err := receipts.ClaimEligible(ctx, time.Now().UTC())
	if err != nil {
		return err
	}

	err = objects.Delete(ctx, bucket, receipt.ObjectKey)
	if err != nil && !errors.Is(err, ErrNotFound) {
		return receipts.MarkReview(ctx, receipt.ID, fmt.Sprintf("delete %s: %v", receipt.ObjectKey, err))
	}

	// An already-absent object is acceptable only after the row was claimed.
	return receipts.MarkDeleted(ctx, receipt.ID, time.Now().UTC())
}
```

The `ErrNotFound` branch is a policy decision, not a universal truth. It is reasonable when reconciliation can prove that the key belonged to this receipt and no replacement is allowed; it is not reasonable when an absent object could mean the key was overwritten or the account boundary was wrong. In that case, send the row to `review` and preserve the evidence.

Presigned downloads follow the same separation. The read handler loads the receipt by account and ID, checks its state is readable, creates a GET capability for the stored key, and returns the URL without persisting it. Set an expiry short enough for the screen's job. A customer who needs a new link should request a new link, not resurrect an old one.

## A Go worker for repeatable receipt deletion

The buy-versus-build choice is mostly an on-call and control-plane decision. A managed object service can reduce the work of disks, replication, and restores; a self-hosted service can offer more direct control but makes those responsibilities yours. Either option still leaves the application responsible for tenant isolation, retention policy, deletion evidence, and authorization.

| Boundary | Useful when | The catch |
| --- | --- | --- |
| Managed object storage | The team wants storage operations outside its primary on-call rotation | Account policy, access configuration, and deletion semantics still need review |
| Self-hosted object storage | The team can staff capacity, replication, upgrades, and restore testing | The platform team owns every storage failure and recovery exercise |
| Media-focused service | Transformations and delivery workflows are central to the product | The service's retention and export model may not match an audit archive |
| Database-only blobs | Files are small and transactional coupling is the dominant requirement | Large images can increase database backup and restore pressure |

The recommendation is not suitable when regulations require immutable retention, version history, or object-lock semantics that the selected storage boundary does not provide. Choose a system with those controls, or keep the retention archive in a separately governed system. Likewise, a public gallery should use a public delivery design; forcing expiring private links onto public content creates needless authorization work.

Your mileage may vary on the exact expiry window. Test it against the client experience, replay risk, and the time needed to download a large original. Do not let a convenient browser URL become the retention policy by accident.

## Where private buckets and presigned downloads stop fitting

The catch is that private, expiring access is the wrong tool for a public gallery or a workflow that needs durable public URLs. Use a public delivery architecture when the content is intentionally public. It is also not the right tool when the retention obligation requires immutable history, version recovery, or object-lock semantics that the selected storage boundary does not provide; use a separately governed archive in that case.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
