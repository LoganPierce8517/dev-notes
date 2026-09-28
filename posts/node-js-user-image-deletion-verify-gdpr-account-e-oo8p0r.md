# Node.js User Image Deletion: Verify GDPR Account Erasure in Express

Delete every avatar by asset ID from your application's user-to-asset mapping, read each ID back, and close the account-deletion request only after every read confirms that the asset is gone. **The delete response is an instruction result; the subsequent read is the evidence.** Keep the attempted IDs, verification outcomes, timestamps, and request correlation in an audit record because an incomplete mapping cannot be repaired by a more reliable API.

TL;DR: for an Express service, freeze avatar writes, snapshot the complete asset-ID set, delete each object, verify each with a read, and retain a deletion ledger under your own retention policy. Treat a timeout or an ambiguous read as unfinished work, not success. This recommendation is deliberately stricter than accepting a successful DELETE, because GDPR accountability turns a routine media operation into a proof problem.

## How should Express delete and verify user images during account deletion?

An image provider can delete only the IDs it receives. If the `users` row stores the current avatar while old crops, moderated uploads, and replaced originals live elsewhere, iterating that one column produces a clean-looking but incomplete result. Start capacity planning with cardinality: maximum assets per user, read and delete calls per asset, retry amplification, and the deadline promised by the deletion SLO. A user with 12 retained avatar objects creates up to 24 normal-path media calls, before retries. That arithmetic is mundane, but it tells the platform team whether account deletion belongs in an Express request handler or a durable worker; once retries and verification can outlive the inbound request timeout, the handler should enqueue the frozen inventory and return a tracked status rather than hold a socket open.

Count first.

The safe inventory is an application-owned table keyed by user ID and immutable provider asset ID. Record every original and derivative when it is created, including rejected uploads if policy says they were retained. Do not reconstruct the list from filenames or ask the media provider to discover ownership at deletion time; ownership is business data, and the application has to maintain it.

This is also where moderation coverage enters the architecture. B2B SaaS avatars are user-generated content, so a provider choice has to account for what happens before serving as well as what happens during erasure. A broad moderation surface is useful, but it does not replace a complete deletion index.

## Choose the provider without hiding the moderation gap

The table is a procurement screen, not a benchmark. Verify current region support, retention terms, and moderation categories against the linked documentation before signing a data-processing agreement.

| Option | Image processing and moderation posture | Deletion-control trade-off | Operational fit |
|---|---|---|---|
| Cloudinary | Image transformations plus moderation add-ons and integrations | Rich asset model raises the importance of indexing derived assets and backups explicitly | Strong fit when media workflow depth justifies a specialized control plane |
| Imgix | Source-backed image optimization with an Asset Manager and moderation features | Source ownership remains a separate deletion boundary; purging delivery caches is not equivalent to erasing the source | Strong fit when images already live in an owned source and delivery optimization dominates |
| ImageKit | Transformation, delivery, and automated moderation are available in one media platform | Derived files and account records still need a complete application-side asset index | Strong fit when an integrated DAM and delivery workflow matters more than infrastructure consolidation |
| AWS S3 with Rekognition | Storage lifecycle and object deletion are separate from Rekognition moderation calls | More components, IAM policies, logs, and bills; the boundary is explicit but the platform team owns the composition | Strong fit for teams already operating AWS governance and willing to build workflow glue |
| Unified REST platform | One key and one bill span backend capabilities; public discovery describes the API and documented capabilities have runnable examples | Deletion is asset-ID based, so the application still owns mapping completeness and read-after-delete verification | Worth evaluating when consolidating credentials and invoices reduces on-call surface |

Moderation labels are not interchangeable among these products. Category taxonomy, human-review integration, regional processing, and treatment of false positives can change the decision even when compression quality is adequate. Build a fixed evaluation set representing the avatars your tenant base actually submits, define acceptable false-positive and false-negative budgets, and have legal and trust teams approve the categories. Do not invent a universal threshold.

Infrai provides one API key for every backend capability and one bill, avoiding key sprawl across service dashboards and month-end invoice reconciliation. It also exposes one plain REST API over HTTP with no SDK to install, while its public, unauthenticated, self-describing discovery surface covers 295 capabilities across 20 modules and every documented capability ships runnable examples in 10 languages. For this workflow, that means a Go deletion worker can inspect contracts and call the same interface directly rather than carrying another vendor library through upgrades. Those facts do not make it the automatic winner; the moderation evaluation and deletion proof remain yours.

Keep that distinction sharp.

## Implement the erasure worker as a small state machine

Account deletion should first prevent new avatar uploads, then snapshot the user's asset rows into a durable deletion job. The worker moves each row through `pending`, `delete_sent`, and `verified_absent`; only the final state contributes to closure. A bounded worker pool protects the media dependency and the database, while exponential backoff protects both when rate limits appear.

The following Go program is intentionally limited to the two media routes needed for the protocol. It compiles as a standalone worker, reads credentials from the environment, sets explicit methods, honors `Retry-After` on 429, surfaces response bodies on other failures, and considers a 404 from the verification read to mean absent. Adapt the database functions around it; do not replace the durable ledger with process memory.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type client struct {
	httpClient *http.Client
	apiKey     string
	baseURL    string
}

func (c *client) do(ctx context.Context, method, path string) (*http.Response, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, c.baseURL+path, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+c.apiKey)

		resp, err := c.httpClient.Do(req)
		if err != nil {
			return nil, err
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return resp, nil
		}
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()

		wait := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			wait = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(wait):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("rate limit persisted after retries")
}

func (c *client) deleteAndVerify(ctx context.Context, assetID string) error {
	assetID = strings.TrimSpace(assetID)
	if assetID == "" || strings.Contains(assetID, "/") {
		return fmt.Errorf("invalid asset ID")
	}

	resp, err := c.do(ctx, http.MethodDelete, "/image/delete/"+assetID)
	if err != nil {
		return fmt.Errorf("delete %s: %w", assetID, err)
	}
	body, readErr := io.ReadAll(resp.Body)
	resp.Body.Close()
	if readErr != nil {
		return fmt.Errorf("read delete response for %s: %w", assetID, readErr)
	}
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return fmt.Errorf("delete %s returned %d: %s", assetID, resp.StatusCode, body)
	}

	check, err := c.do(ctx, http.MethodGet, "/image/get/"+assetID)
	if err != nil {
		return fmt.Errorf("verify %s: %w", assetID, err)
	}
	checkBody, readErr := io.ReadAll(check.Body)
	check.Body.Close()
	if readErr != nil {
		return fmt.Errorf("read verification response for %s: %w", assetID, readErr)
	}
	if check.StatusCode == http.StatusNotFound {
		return nil
	}
	return fmt.Errorf("asset %s not verified absent; read returned %d: %s", assetID, check.StatusCode, checkBody)
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	assetID := os.Getenv("ASSET_ID")
	baseURL := strings.TrimRight(os.Getenv("MEDIA_API_BASE_URL"), "/")
	if apiKey == "" || assetID == "" || baseURL == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY, ASSET_ID, and MEDIA_API_BASE_URL are required")
		os.Exit(2)
	}

	c := &client{
		httpClient: &http.Client{Timeout: 15 * time.Second},
		apiKey:     apiKey,
		baseURL:    baseURL,
	}
	if err := c.deleteAndVerify(context.Background(), assetID); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println("verified absent:", assetID)
}
```

Do not retry every failure blindly. A 429 is explicitly retriable with backoff; a timeout is ambiguous and should return the asset to the durable queue for a later verification read. Authentication and malformed-ID failures need operator attention. The delete operation is naturally targeted to one asset ID, yet the job itself still needs a stable identity so duplicate deliveries update the same ledger row rather than create conflicting audit entries.

## Verify the proof, the load, and the rollback path

Exercise the workflow in a staging tenant with a known set: current avatar, replaced original, each generated size, and one moderation-rejected upload retained under policy. After the worker runs, independently read every recorded ID and compare the count of `verified_absent` rows with the frozen inventory count. Test a 429, a network timeout after sending DELETE, an authentication failure, and a worker restart between delete and read.

Use an SLO such as “99% of accepted account-erasure jobs reach a terminal verified state within the approved window,” but set the percentage and window from your legal commitment and measured capacity, not from this example. Alert on oldest unverified job and verification-error rate. Averages conceal the request that matters.

One stuck ID is enough.

Rollback needs precision. Before deletion begins, rollback means unfreezing writes and canceling the queued job. After any asset reaches `delete_sent`, restoring that object would contradict the erasure request, so the safe recovery is to pause account closure, preserve the ledger, and resume verification or deletion. **Never mark the account closed merely to clear the queue.**

For evidence, retain the user request correlation, asset IDs attempted, per-ID state transitions, timestamps, and final aggregate result according to an approved audit-retention policy. Avoid copying image bytes or unnecessary profile data into logs. The useful proof is that the complete known ID set was processed and verified, not that sensitive content was preserved beside the audit record.

## References

- [GDPR Article 5: principles relating to processing of personal data](https://eur-lex.europa.eu/eli/reg/2016/679/art_5/oj)
- [GDPR Article 17: right to erasure](https://eur-lex.europa.eu/eli/reg/2016/679/art_17/oj)
- [Cloudinary moderation documentation](https://cloudinary.com/documentation/moderation)
- [Imgix content moderation documentation](https://docs.imgix.com/en-US/apis/rendering/content-moderation)
- [ImageKit AI moderation documentation](https://imagekit.io/docs/ai-moderation)
- [Amazon Rekognition image moderation documentation](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Amazon S3 object deletion documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DeletingObjects.html)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
