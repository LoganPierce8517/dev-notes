# Next.js API route rules for object storage PNG download links

Short answer: put each tenant's generated images in the storage region assigned to that tenant, keep the objects private, and issue an expiring download authorization from a server-side storage boundary rather than exposing a bucket path from a Next.js handler.

The least complex design is a private bucket per residency boundary, a durable database record for the object key and region, and a short-lived read authorization generated only after the application has authenticated the caller. It keeps the API route small, gives an on-call engineer one place to reason about access, and avoids treating a URL as either an identity system or a permanent asset identifier.

Keep the link disposable.

## The incident lesson: an image URL is not a storage contract

Generated PNGs tend to arrive in a product as an apparently small feature: call a model, save its bytes, show a download button. The operational problem begins once retries, tenant moves, support tooling, cache behavior, and residency obligations all touch that same object. A URL copied into a ticket can outlive the session that created it; a retry can repeat a write; a regional fallback can become a data-placement decision that nobody reviewed. Those are different classes of failure, but they share one cause: the application has not separated object identity, authorization, and location.

Keep those three records distinct. Object identity is an internal key such as `tenants/acme/images/01J...png`. Authorization is an expiring, signed read request or an application download endpoint that verifies the user. Location is the tenant's assigned storage region, stored with the tenant rather than inferred from a browser header, an IP address, or a request-time preference. The browser may receive an authorization URL; it should not receive credentials or an object-store namespace that it can enumerate.

This is an SLO concern, not paperwork. If the image-download availability objective is 99.9%, the dependency budget includes the app's authorization path, the chosen storage region, DNS, and any CDN in front of private content. Count them before adding a second redirect. A signed URL can reduce application read load, but it does not remove the need to observe authorization failures, expiry rejections, object-not-found results, download latency, and the gap between database records and stored objects.

One rule helps during an incident: an object key should be sufficient to locate the record, while a link should be disposable.

## How should a Next.js API route store AI-generated PNGs for US and EU SaaS tenants?

Make the Next.js API route an authentication and orchestration boundary. It obtains the tenant from the authenticated principal, looks up that tenant's residency assignment, validates that it received PNG bytes, and hands the bytes to a storage service. The storage service chooses the bucket or endpoint from the recorded region, writes the object with `image/png`, and persists its key with the generation record. The route does not select a region from client input.

The same boundary should mint downloads. On a request for an image, first check that the caller may read the generation record. Then ask the storage adapter for a time-limited read URL, or stream the object through the application when audit logging, response transformation, or a tighter policy requires it. The signing syntax is provider-specific, which is exactly why it belongs behind a narrow interface. Do not scatter SDK calls through route handlers and background workers; a future move, a test double, or a change in signing policy should affect one adapter.

For a private response that must not be retained by shared caches, `Cache-Control: private, no-store` expresses the intent that it is for one user and should not be stored. The correct policy depends on the response type: a redirect, a streamed download, and an immutable public asset do not have the same cache contract. The cache directive cannot repair an over-broad authorization decision.

Here is the preventive path in Go. It is deliberately an interface rather than a vendor tutorial: the important contract is that region selection is server-controlled and the returned URL has a bounded lifetime.

```go
package images

import (
	"context"
	"fmt"
	"time"
)

type Region string

const (
	RegionUS Region = "us"
	RegionEU Region = "eu"
)

type ObjectStore interface {
	Put(ctx context.Context, region Region, key string, body []byte, contentType string) error
	SignedDownload(ctx context.Context, region Region, key, filename string, ttl time.Duration) (string, error)
}

type Tenant struct {
	ID     string
	Region Region
}

func StorePNG(ctx context.Context, store ObjectStore, tenant Tenant, key string, png []byte) error {
	if tenant.Region != RegionUS && tenant.Region != RegionEU {
		return fmt.Errorf("unsupported residency region %q", tenant.Region)
	}
	if len(png) == 0 {
		return fmt.Errorf("empty PNG")
	}
	return store.Put(ctx, tenant.Region, key, png, "image/png")
}

func DownloadURL(ctx context.Context, store ObjectStore, tenant Tenant, key string) (string, error) {
	return store.SignedDownload(ctx, tenant.Region, key, "generated-image.png", 10*time.Minute)
}
```

The `key` above must come from a record the application owns, not directly from an arbitrary query parameter. A common design uses a generated image ID in the route, fetches its tenant-scoped record, and uses the stored object key. For retryable generation jobs, retain an idempotency key or job identifier in that record so a repeated job can resolve to the intended object instead of creating an unrelated object merely because the first response was lost.

## Capacity planning and the write path

PNG storage is easy to under-budget because average file size hides the tail. Plan from observed distributions: p50 and p95 image sizes, generations per tenant, retry rate, retention duration, and the peak concurrent writes during model backlogs. Then calculate object count as well as bytes. Listing, lifecycle scans, reconciliation, and metadata operations can care far more about count than total capacity.

The practical review is a queueing exercise before it is a storage procurement exercise. Start with a peak arrival rate for completed generations, multiply it by the high-percentile image size rather than the mean, and compare the resulting write bandwidth with the limits of the chosen regional path. Then run the same calculation for a backlog release, where completed jobs may arrive in a short burst after an upstream dependency recovers. A design that handles ordinary traffic but starts timing out during that burst can cause clients and workers to retry, which adds pressure precisely when capacity is already tight. Put bounded retries around the generation job, not blind retries around every object write, and emit an idempotency result that lets a worker distinguish a completed image from work that was never recorded. Don't declare capacity solved until the test environment has exercised that burst with authorization and metadata writes enabled; a benchmark that uploads anonymous files misses the application path that users actually depend on.

The database is the control plane. Store a tenant ID, region, object key, content type, byte count, creation time, retention state, and the generation or idempotency identifier that created the image. Avoid treating object-listing as the source of truth for an end-user gallery. Prefix listings are useful for reconciliation, but they are a weak substitute for an access-controlled application index.

Reconciliation should be a scheduled, bounded process: identify database rows whose objects cannot be read, objects with no corresponding row after a grace period, and records that have passed a documented retention policy. Record counts and ages, page the owning service only on a defined threshold, and test the process against a non-production bucket before relying on it. A deletion job with no tenant filter is the kind of capacity optimization that becomes a postmortem.

Use a content-addressed key only when duplicate bytes are interchangeable under the product's access model. It can reduce duplicate storage and make retried writes converge, but it may leak equality information if keys are observable, and it complicates per-object deletion when one tenant must be removed independently. An opaque, tenant-scoped key plus an explicit idempotency record is often easier to operate. There is no universal winner.

## Buy, build, or keep bytes close to the database

The buy-versus-build choice is mostly a pager assignment. Managed object storage moves media durability and disk replacement away from the application team, while the team still owns identity, retention, encryption choices, recovery testing, and a correct regional mapping. A self-hosted object service can fit an existing operations practice, but it makes replication lag, upgrade planning, capacity headroom, and restore drills part of the service objective. Storing binary images directly in a relational database can be appropriate for small, tightly transactional artifacts; it also expands backup size and changes restore time in ways that need explicit testing.

| Option | Operational owner | Useful when | Boundary to accept |
| --- | --- | --- | --- |
| Managed object storage | Provider for media durability; application team for access policy | Image volume is variable and the team wants managed durability | Regional and egress behavior must be reviewed against the product's data policy |
| Self-hosted object service | Platform team | Storage operations and recovery drills are already a staffed responsibility | Replication, upgrades, capacity, and restore verification join the on-call load |
| Relational database blobs | Application and database teams | Images are small, few, and need the same transaction as nearby metadata | Backups, replication, and recovery time grow with the media |

The catch is that a signed download pattern is not suitable for public, immutable images that must be heavily cached. Those assets usually need a deliberately public publishing path, cacheable URLs, and separate controls for who may publish them. Conversely, do not use a public prefix for user-generated or generated outputs merely because it makes the front end convenient. If an organization cannot operate a reliable regional object layer, a managed service in the approved residency region is generally the lower-risk operational choice; if the organization needs data handling that its provider cannot contractually or technically support, retain the bytes in an environment it controls and accept the operational cost.

## Test the policy, not only the upload

Most happy-path tests prove that bytes can be written. The test matrix needs to prove that the policy survives the awkward paths: a US tenant cannot request an EU object's authorization, an expired link is rejected, a user from another tenant cannot name a key into existence, a retried generation resolves through its idempotency record, and a retention worker never deletes an active record. Add a clock-controlled test around URL expiry and a contract test for the storage adapter, because signing behavior and response headers sit at a boundary where small differences matter.

For deploys, expose the storage adapter's region choice, write latency, authorization failures, bytes written, and reconciliation discrepancy as metrics with tenant identifiers handled carefully. Keep logs useful without putting signed URLs or sensitive prompts into them. Test recovery with a representative sample of records and objects, then measure the time to rebuild the application index and the time to serve authorized downloads again. Those measurements, rather than a storage feature checklist, are what turn a design into an operational promise.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://cloud.google.com/storage/docs
- https://www.rfc-editor.org/rfc/rfc9110.html
