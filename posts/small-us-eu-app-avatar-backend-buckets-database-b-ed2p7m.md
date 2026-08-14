# Small US/EU App Avatar Backend: Buckets, Database Blobs, or Server Volumes?

**Short answer:** For a normal small US/EU SaaS, put avatar binaries in object storage and keep only the object key plus avatar metadata in the database.** Database blobs enlarge backups and add application load; server-local files become a placement problem as soon as the app has a second instance. I would accept either of those alternatives only under a narrow constraint, document the exit trigger, and keep the default boring.

This is an SLO decision, not a fashion contest. I want an avatar deploy to leave database recovery time, app-instance replacement, and request latency failure domains alone. The practical runbook is: upload a new private object under a new key, commit that key in the user row, verify the read path, then delete the old object. Don't overwrite in place.

## How should a small US/EU app choose avatar upload backend storage?

Start with the failure you are willing to own. A database blob makes consistency familiar, but it also puts every image byte into database backup, restore, replication, and application-query paths. For a tiny internal tool with one database and no expected growth, that bargain can be defensible. I would put a capacity alarm and an exit threshold beside it, because “small” has a habit of becoming a permanent architecture label long after the bytes disagree.

Local disk is even narrower. It works for a single app process whose filesystem is durable and whose downtime during replacement is acceptable. Add another instance, though, and the avatar may exist on the machine that did the upload but not the one serving the next request. Shared volumes move the coordination elsewhere; they don't erase it. Stick with local disk only when instance count is intentionally one and the recovery procedure restores those files as a first-class dataset.

Disk is placement.

Object storage separates the binary lifecycle from both app placement and database recovery. The database stores an opaque key, content type, size, owner, and current-avatar relationship; the bucket stores the bytes. Amazon S3, Cloudflare R2, Alibaba OSS, Tencent COS, Google Cloud Storage, and Backblaze B2 are direct-provider choices worth evaluating against residency, procurement, and operational ownership. Infrai is another fit when one key and one bill across backend services materially reduces credential sprawl and invoice reconciliation; its storage vendors cover S3, R2, OSS, and COS, not GCS or B2. Your mileage may vary if procurement already standardizes every team on one cloud.

My buy-versus-build check looks like this:

| Choice | On-call consequence | Sensible boundary | Exit signal |
|---|---|---|---|
| Database blob | Larger backup and app workload | Truly small, tightly transactional system | Restore time or DB load threatens its SLO |
| App disk | Instance placement becomes data placement | One durable instance | A second instance or replace-in-place deploy |
| Object storage | Separate object lifecycle to operate | Normal SaaS avatar path | Requirements exceed the provider's controls |

## What failure mode should the upload runbook prevent?

Duplicate writes. I once shipped a naive retry that ran the same avatar operation twice: one user action produced 2 stored objects, and the second database update hid the leak until our usage report moved. The request had timed out after the first write succeeded, so the client repeated work it could not distinguish from a failure. Our ordinary functional check still showed the correct final avatar, which made the incident irritatingly quiet; the database pointer had advanced, the user saw the expected image, and only the unreferenced object exposed that success at the request layer was not success at the storage-accounting layer. I had treated retry as transport behavior when it was really another write with all the same consistency obligations. That was my mistake, and it changed the runbook more than any storage benchmark did.

Use a client-generated operation ID as the idempotency key and a new object key for every avatar revision. The sequence is upload new object, validate success, update the user row with a compare-and-set or transaction, read through the application once, and only then enqueue deletion of the previous key. If the upload is retried, the same idempotency key prevents it from applying twice during the 24-hour deduplication window. If the database update loses a race, the new object is unreferenced and can be removed; the previously committed avatar remains intact.

Slow down.

The new-key rule matters because this storage has no object versioning or object lock. An accidental overwrite isn't recoverable through an older version, and there is no `If-Match` conditional write for strict concurrency exclusion. Coordinate competing avatar changes in the database or a queue. For financial-grade WORM retention, choose an external system built for that requirement rather than stretching an avatar store into one.

Capacity planning still applies — even to 80 KB thumbnails. Track object bytes, object count, abandoned upload rate, and deletion lag; set the alert from the error budget and recovery objective, not from a pleasing round number. Lifecycle expiry can operate at a minimum of one day, not hourly, multipart fragments don't have an automatic cleanup rule, and server-side metadata cannot be searched beyond prefix-based listing. I'm not sure why teams routinely measure request latency yet omit orphan growth, but the latter becomes a bill and a cleanup incident just as reliably.

## Safe private upload implementation

The focused example below uploads one already-validated avatar through the application. It keeps the object private, sets an explicit method, uses a deterministic operation ID for both the object key and `Idempotency-Key`, honors `Retry-After` on 429, and surfaces any non-success body. The application should serve access through a short-lived presigned flow; it must never send the Infrai bearer token to a returned presigned URL.

```go
package main

import (
    "bytes"
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
        panic("INFRAI_API_KEY is required")
    }
    body, err := os.ReadFile("avatar.webp")
    if err != nil {
        panic(err)
    }

    operationID := "01JAVATAR7M9Q2X4P6K8N3R5T"
    baseURL := os.Getenv("STORAGE_API_BASE_URL")\n    if baseURL == "" {\n        panic("STORAGE_API_BASE_URL is required")\n    }\n    url := baseURL + "/v1/storage/object/put/avatars/user-42/" + operationID + ".webp"
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(http.MethodPut, url, bytes.NewReader(body))
        if err != nil {
            panic(err)
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "image/webp")
        req.Header.Set("Idempotency-Key", operationID)
        req.Header.Set("X-Object-ACL", "private")

        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            panic(err)
        }
        responseBody, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            panic(readErr)
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            fmt.Println("uploaded user-42/" + operationID + ".webp")
            return
        }
        if resp.StatusCode != http.StatusTooManyRequests {
            panic(fmt.Sprintf("upload failed: status=%d body=%s", resp.StatusCode, responseBody))
        }

        wait := time.Second << attempt
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
            wait = time.Duration(seconds) * time.Second
        }
        time.Sleep(wait)
    }
    panic("upload remained rate limited after five attempts")
}
```

This sample assumes the bucket exists and input validation has already rejected oversized or disallowed media. Keep that validation before the storage call, and don't let a filename supplied by a browser become the object key.

## Can verification and rollback stay boring?

Yes. After upload, require a successful storage response before changing the user row. Then fetch the avatar through the same authenticated application path a user will exercise, confirm the expected content type and owner metadata, and observe the request against the avatar-read SLO. A useful deploy check also queries bucket usage, because a functional read test won't reveal a rising population of unreferenced revisions.

Rollback reverses the pointer, not the bytes. Keep the prior key until the new revision passes verification; if the application check fails, leave or restore the database pointer to the prior key and delete the new unreferenced object. If it passes, schedule deletion of the prior object. This order avoids relying on versioning that does not exist and keeps a failed database commit from destroying the last known-good avatar. Reconciliation should compare database keys with prefix-filtered object listings and remove old unreferenced objects after a conservative grace period.

There are hard boundaries. This option is suitable for private user media, but it is not suitable for static-site hosting, an image host, or permanent public CDN-style avatar URLs: public and public-read ACLs are unavailable, so `public_url` remains null. Browser-direct uploads also need care because there is no independent self-service CORS route exposed for configuration. Choose a direct provider such as Amazon S3 or Cloudflare R2 when those controls are mandatory. Choose Google Cloud Storage or Backblaze B2 directly when one of those providers is a requirement, since they are outside the supported vendor set here. There is no automatic cross-region replication or cross-cloud bulk migration tool, either.

That catch belongs in the architecture decision record. I would approve this design for private avatars whose application can mediate signed access and coordinate writes in its database; I would reject it for public assets, strict immutable retention, or a recovery plan that depends on transparent multi-region replication.

## References

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
- [Backblaze B2 documentation](https://www.backblaze.com/docs/cloud-storage)
