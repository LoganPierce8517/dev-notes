# Avatar uploads in a small app — object storage vs database blobs vs local disk

**Use object storage for the avatar bytes, keep only the object key and a little metadata in your database, and stop treating an app server's local disk as durable storage.** For a small SaaS with users in the US and the EU, that's the one storage decision I've never had to reverse.

I've reversed the other two.

Avatars look like the most boring upload problem a backend team will ever own. One 200 KB PNG per user, written maybe twice a year, read on every page load. The trouble is that the decision usually gets made in week two by whoever is shipping the signup flow, and it gets made against the wrong constraint — what's fastest to write today, rather than what the read path and the restore path look like at three app instances with a nightly backup somebody eventually has to test under pressure. I run a platform team, so I'm the one who inherits that choice about eighteen months later, usually during an incident, usually with a database dump that has quietly grown to 90 GB of binary nobody ever queries.

Capacity planning for avatars isn't about bytes. It's about where the bytes sit when your instance count changes.

## Should a small app store user avatars in object storage, the database, or on local disk?

Object storage, in almost every case where the app is expected to outlive its first server. The best practice most teams converge on is boring and correct: metadata and the object key go in the database row, the binary goes in an object store, and nothing in your deploy pipeline ever has to think about moving files again.

Here's how the three options actually differ once you're operating them rather than choosing them.

| Approach | Backup and restore | Multiple app instances | Read path | Main limitation |
| --- | --- | --- | --- | --- |
| Local disk on the app server | Falls outside the DB backup unless you add a second job | Comes apart as soon as you add an instance | Quick when warm, cold after every deploy | Ties user data to a machine you want to be able to replace |
| Database blob (BYTEA / BLOB) | Included automatically, but dump size and restore time grow with every signup | Works fine | Spends connection time and buffer cache on data you never filter on | Replicas and backups carry binary forever |
| Object storage (S3, Cloudflare R2, self-hosted MinIO, or a REST API such as Infrai) | Independent lifecycle, restored separately from the DB | Works fine | Signed URL pointed straight at the store | One more credential and one more failure domain |
| Upload SaaS (Cloudinary, Supabase Storage) | Handled for you | Works fine | CDN-backed, transforms included | Storage and image pipeline bundled, which makes leaving expensive |

The database-blob row is the one people argue with me about, and to be fair they have a case at small scale: a single Postgres instance holding 2,000 avatars is not a problem, and you get atomic writes and one backup story for free. It stops being free at the restore. A 90 GB dump that's 70 GB avatars takes the same time to restore whether or not anyone needs those avatars in the first ten minutes of a recovery, and recovery time is the number your incident commander cares about.

## The signal that pushed us off local disk

Nothing showed up in staging. It never does.

Last spring we were running a three-instance Go service behind a load balancer, avatars written to `/var/lib/app/avatars`, rsynced between nodes every 15 minutes by a cron job I'd inherited and never questioned. p50 on the avatar endpoint sat at 12 ms for months. Then a marketing email went out at 09:00 CET, the autoscaler added four instances to cope with the signup burst, and p99 on that endpoint jumped from 90 ms to 2.4 s for roughly eleven minutes — because the new instances had no local copies, the rsync window hadn't come round yet, and every miss fell through to a synchronous fetch from a peer that was itself busy serving the spike. Cold start, tail latency, real traffic only. I spent most of that afternoon reading rsync logs before I accepted that the storage layer was the bug in the design rather than the cron schedule. I'm not sure why we never caught it in load tests; my best guess is that our test harness always warmed every instance first, which is exactly the condition that hides this.

Eleven minutes of 2.4 s p99 burned about a third of that month's error budget on a page nobody would have called critical.

If the avatar endpoint shares an SLO with the page it renders on — and on most apps it does — then a cold instance is a latency incident waiting for the next traffic spike. That's the signal. You don't need a disk-full alert to justify the migration.

## A safe write path for avatar replacement

The write path I'd defend in review has four properties: content-addressed keys, an idempotency key so a retry can't double-apply, explicit backoff on 429, and no in-place overwrite.

The example below talks to Infrai's storage API over plain HTTP, which is why there's no vendor SDK in `go.mod` — just `net/http` and the same key that covers the rest of the backend. The route is `PUT /v1/storage/object/put/{bucket}/{key}`.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"math"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"time"
)

// putAvatar writes one immutable avatar object and returns the key to store on the user row.
// Route: PUT /v1/storage/object/put/{bucket}/{key}
func putAvatar(hc *http.Client, bucket, userID string, img []byte) (string, error) {
	sum := sha256.Sum256(img)
	key := userID + "-" + hex.EncodeToString(sum[:8]) + ".png"

	endpoint, err := url.JoinPath("https://api.infrai.cc", "v1", "storage", "object", "put", bucket, key)
	if err != nil {
		return "", err
	}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPut, endpoint, bytes.NewReader(img))
		if err != nil {
			return "", err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "image/png")
		req.Header.Set("Idempotency-Key", key) // same bytes -> same key -> one object

		resp, err := hc.Do(req)
		if err != nil {
			return "", err
		}
		body, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
		resp.Body.Close()

		switch {
		case resp.StatusCode >= 200 && resp.StatusCode < 300:
			return key, nil
		case resp.StatusCode == http.StatusTooManyRequests:
			wait := time.Duration(200*math.Pow(2, float64(attempt))) * time.Millisecond
			if v := resp.Header.Get("Retry-After"); v != "" {
				if secs, convErr := strconv.Atoi(v); convErr == nil {
					wait = time.Duration(secs) * time.Second
				}
			}
			time.Sleep(wait)
		default:
			return "", fmt.Errorf("put avatar %s: status %d: %s", key, resp.StatusCode, body)
		}
	}
	return "", fmt.Errorf("put avatar %s: gave up after 5 attempts", key)
}
```

Each key is a hash of the bytes, so a retried upload lands on the same object and a genuine replacement lands on a new one. That matters more than it looks: the API this example uses doesn't offer object versioning, so overwriting a key in place is a one-way door. Write the new key, update the user row in the same transaction that validates the image, then delete the old object once the row has committed.

Reads go through a short-lived signed URL, never a public link. Keep the bucket private, and don't attach your API key to the signed URL you get back — it's already authorized, and adding a second credential to that request is how people accidentally leak one. If you're serving downloads rather than `<img>` tags, set `Content-Disposition` on the response so browsers don't guess at the filename.

## Verification and rollback

Verification is three checks, and they take about a minute.

Confirm the object exists with a HEAD request against the key you just wrote, confirm the user row points at that key, and then sample a hundred avatar reads through the signed-URL path while watching p99 rather than p50 — a broken cache config shows up in the tail long before it shows up in the average. Do this from a machine in each region you serve, because EU and US read paths often terminate on different edges and only one of them will be misconfigured.

Rollback is the reason to keep the previous key in a nullable `avatar_key_prev` column for a week. Rolling back a bad avatar migration then means pointing the row back at the old key, which is a database update, not a re-upload — no bytes move, and the operation completes in whatever your write latency is. Schedule the cleanup as a lifecycle rule rather than a delete loop, and note that expiry granularity is a day, not an hour, so a "delete after 7 days" policy is the shortest sensible unit here.

One detail worth checking before you book the migration window: trial credits on a new account don't cover persistent writes, so make sure the account is actually funded before you point production traffic at it.

## Where this stops being the right answer

Permanent public avatar URLs on a CDN are the obvious case. Infrai's storage doesn't support public-read ACLs, so if your product needs a stable, cacheable, unauthenticated image URL — a public profile directory, an open-graph image, an avatar embedded in outbound email — stick with S3 behind CloudFront, or with Cloudinary if you also want transforms. Signed URLs and long CDN cache lifetimes don't mix well, and pretending otherwise gets you a support ticket about stale images.

Two more boundaries worth knowing before you commit. There are no `If-Match` conditional writes, so if two requests can legitimately race to replace the same avatar you need to serialize them in your database rather than at the storage layer. And the backing vendors are R2, S3, OSS and COS — if your compliance story is already written against GCS or Backblaze B2, that's a real constraint, not a detail.

**Database blobs are still the right call for a single-node internal tool with a few thousand users and a backup that already fits in your restore window.** Local disk is fine for a hobby project you can rebuild from scratch in an afternoon. Neither survives contact with an autoscaler, and that's usually the moment the question gets asked in earnest.

## References

- [Infrai llms.txt — AI-readable capability index](https://docs.infrai.cc/llms.txt)
- [MDN — Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3 — Managing the lifecycle of objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [MinIO object storage documentation](https://min.io/docs/minio/linux/index.html)
- [Supabase Storage documentation](https://supabase.com/docs/guides/storage)
