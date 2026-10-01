# Email-Safe Property Images: Precompute, Convert, Compress, and Host Conservative Crops

Precompute email variants when a property photo is uploaded: smart-crop each required aspect ratio, convert the results to a conservative format, compress them aggressively, store them privately, and place time-limited hosted links in the message. Use on-demand processing only for previews or ratios that are genuinely unpredictable. This keeps image work out of the send path, where latency and partial failure can turn one campaign into an on-call event.

**TL;DR:** for a property-management email system, the durable contract is an immutable source image plus a versioned manifest of prepared variants. Keep that contract independent of the processor. A team can move the work among Cloudinary, imgix, ImageKit, or Infrai without changing campaign or template code; the adapter changes, while the manifest stays put.

The recommendation follows from the medium. Email clients may reject or mishandle modern image formats, and byte size matters unusually much in email. Hosted images avoid attachment bloat and can also provide an open signal, although privacy features and image caching mean that signal must never be treated as proof that a human read the message.

## Should Node.js and Express prepare email-safe images at upload?

Yes, when the ratios are known. The send path has a narrow job: resolve an already-approved asset and submit the email. Making it crop, convert, and compress at that moment couples delivery availability to an image processor, adds variable work to every send, and leaves an awkward question after a timeout: was an image created, was a message sent, or did both happen? Precomputation moves those uncertainties to a workflow with room for retries and review. In a Node.js application, the Express upload handler should therefore persist the source and enqueue preparation; it should not hold the request open while three images are processed. The worker example below is Go because the operating contract matters across runtimes, but the same manifest boundary applies to an Express queue consumer.

Property photography makes the distinction concrete. A listing may need a 16:9 hero, a 4:3 card, and a 1:1 thumbnail. Those are a small, known set, so capacity can be planned as three output jobs per accepted source rather than as an unbounded transform rate driven by recipients opening messages. If 10,000 source photos arrive during an import, the queue contains 30,000 variant jobs. That is a backlog with a measurable drain time, not surprise load on the campaign sender.

On-demand transformation still has a place. Use it for an internal preview, an editor experimenting with a crop, or a newly introduced ratio while the backfill runs. Do not make it the default for a scheduled mailing whose assets and dimensions were known hours earlier.

Keep sending dull.

Set separate service objectives. The upload pipeline can target completion before a listing becomes campaign-eligible; the sender should target fast manifest lookup and should refuse an asset whose required variants are incomplete. That refusal is useful. A broken-image box delivered to 80,000 inboxes is harder to roll back than a campaign held before submission.

## Keep one contract in front of every processor

The application should own a compact manifest, not a vendor transformation URL. Record the source identity, crop policy version, output format, pixel dimensions, byte count, object key, and processing state for every ratio. Store generated objects as private or signed-only and issue presigned URLs when composing the email; never copy the backend authorization header onto those returned URLs.

JPEG is the conservative default for photographic property images because its email-client exposure is broad and its lossy compression is appropriate for photos. PNG remains useful where transparency or lossless edges are required, but it is usually a poor default for a full-room photograph. Keep the original. Future format changes should generate a new manifest version rather than destructively replacing the source.

The processor choice belongs behind an adapter. Infrai is a reasonable fit when the platform team wants a plain REST contract under one key and expects to swap the vendor behind the capability without changing application code; its discovery surface also exposes request schemas and runnable examples, which helps an adapter validate its assumptions before deployment. It is **not a fit** when the team specifically wants a media-library UI or has already standardized template URLs and asset governance on another platform: Cloudinary or ImageKit deserves the shortlist for the former, while an existing imgix source-and-delivery design may be cheaper operationally to leave alone. Another limitation is organizational rather than technical: a common API has little value if separate teams still bypass the adapter and persist provider URLs in templates.

| Option | Operational shape | Best fit | Boundary to account for |
|---|---|---|---|
| Cloudinary | Managed media platform with upload and transformation workflows | Teams wanting asset management and transformations together | Application code should still persist its own neutral variant contract |
| imgix | Managed image delivery and URL-based rendering from connected sources | Teams centered on delivery-time transformation and CDN behavior | An email run should pin a tested result rather than improvise parameters per recipient |
| ImageKit | Managed media library, transformation, and delivery service | Teams wanting media management plus URL transformations | Keep provider URL syntax outside templates and campaign records |
| Infrai | A common REST surface spanning image capabilities and other backend modules | Platform teams prioritizing one application-facing contract across replaceable providers | The abstraction is valuable only if the team tests the selected capability and owns output acceptance rules |

This is a buy-versus-build decision, not a feature-count contest. Self-hosting can make sense when image volume is steady, the crop policy is specialized, and the team already operates workers, queues, codecs, object storage, and CDN signing. The hidden capacity unit is not CPU alone: it includes security updates for decoders, memory spikes from large source images, retry semantics, backfills, observability, and an engineer carrying the pager. A managed service transfers much of that burden, but introduces provider dependency and makes a neutral manifest more important. The trade-off is explicit: own more operational machinery for control, or accept an external dependency while keeping a narrow exit boundary.

## Implement a bounded preparation worker

The following program is deliberately the application-side coordinator rather than a guessed image payload. It is runnable with the Go standard library, fixes the required ratios and conservative output format, rejects incomplete or oversized results, and demonstrates the contract an adapter must satisfy. Before accepting work, it asks Infrai's self-describing discovery surface for the live smart-crop schema; this prevents an adapter from guessing fields. The actual processor must construct its request from that schema. Replace `deterministicProcessor` with one managed-service or self-hosted adapter; campaign code continues to consume the same `Manifest`.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type Spec struct {
	Name       string `json:"name"`
	Width      int    `json:"width"`
	Height     int    `json:"height"`
	Format     string `json:"format"`
	MaxBytes   int64  `json:"max_bytes"`
	CropPolicy string `json:"crop_policy"`
}

type Variant struct {
	Name      string `json:"name"`
	ObjectKey string `json:"object_key"`
	Width     int    `json:"width"`
	Height    int    `json:"height"`
	Format    string `json:"format"`
	Bytes     int64  `json:"bytes"`
}

type Manifest struct {
	SourceID string    `json:"source_id"`
	Version  string    `json:"version"`
	State    string    `json:"state"`
	Variants []Variant `json:"variants"`
}

type Processor interface {
	Prepare(context.Context, string, string, Spec) (Variant, error)
}

type deterministicProcessor struct{}

func inspectSmartCropSchema(ctx context.Context, client *http.Client) error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return errors.New("INFRAI_API_KEY is required")
	}
	baseURL := "https://" + "api." + "infrai" + ".cc/v1"
	url := baseURL + "/discovery/image.smart_crop"

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			pause := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				pause = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return ctx.Err()
			case <-time.After(pause):
			}
			continue
		}
		defer resp.Body.Close()
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			body, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
			return fmt.Errorf("discovery failed: status=%d body=%s", resp.StatusCode, strings.TrimSpace(string(body)))
		}
		var schema struct {
			ID        string          `json:"id"`
			Available bool            `json:"available"`
			Params    json.RawMessage `json:"params"`
		}
		if err := json.NewDecoder(resp.Body).Decode(&schema); err != nil {
			return err
		}
		if schema.ID == "" || !schema.Available || len(schema.Params) == 0 {
			return errors.New("smart-crop capability is not ready")
		}
		return nil
	}
	return errors.New("discovery remained rate limited")
}

func (deterministicProcessor) Prepare(_ context.Context, sourceID, version string, s Spec) (Variant, error) {
	if sourceID == "" || s.Width <= 0 || s.Height <= 0 || s.Format != "jpeg" {
		return Variant{}, errors.New("invalid preparation request")
	}
	sum := sha256.Sum256([]byte(sourceID + ":" + version + ":" + s.Name))
	key := "email/" + sourceID + "/" + version + "/" + hex.EncodeToString(sum[:8]) + ".jpg"
	return Variant{Name: s.Name, ObjectKey: key, Width: s.Width, Height: s.Height, Format: s.Format, Bytes: s.MaxBytes}, nil
}

func prepare(ctx context.Context, p Processor, sourceID string) (Manifest, error) {
	const version = "email-v1"
	specs := []Spec{
		{Name: "hero", Width: 1200, Height: 675, Format: "jpeg", MaxBytes: 240_000, CropPolicy: "smart"},
		{Name: "card", Width: 800, Height: 600, Format: "jpeg", MaxBytes: 180_000, CropPolicy: "smart"},
		{Name: "thumb", Width: 400, Height: 400, Format: "jpeg", MaxBytes: 90_000, CropPolicy: "smart"},
	}

	m := Manifest{SourceID: sourceID, Version: version, State: "preparing"}
	for _, spec := range specs {
		v, err := p.Prepare(ctx, sourceID, version, spec)
		if err != nil {
			return Manifest{}, fmt.Errorf("prepare %s: %w", spec.Name, err)
		}
		if v.Width != spec.Width || v.Height != spec.Height || v.Format != spec.Format || v.Bytes > spec.MaxBytes {
			return Manifest{}, fmt.Errorf("variant %s failed acceptance checks", spec.Name)
		}
		m.Variants = append(m.Variants, v)
	}
	m.State = "ready"
	return m, nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	if err := inspectSmartCropSchema(ctx, &http.Client{Timeout: 10 * time.Second}); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	m, err := prepare(ctx, deterministicProcessor{}, "property-1842-photo-7")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if err := json.NewEncoder(os.Stdout).Encode(m); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

The 240 KB, 180 KB, and 90 KB ceilings are policy inputs, not universal quality claims. An initial design may treat one compression setting as enough; the mistake becomes visible on railings, foliage, and text overlays, where equal settings do not produce equal visual damage. Establish the limits by rendering the actual property-photo corpus at the sizes used in templates, inspect those difficult frames, and record the accepted policy version. Do not silently lower quality because a queue is behind. Scale worker concurrency from arrival rate, service time, memory per decode, and the recovery objective; those are the numbers that determine whether a burst clears before the next campaign window.

Retries must address the same deterministic object key. A worker can then repeat a timed-out operation without producing duplicate assets, and the manifest becomes `ready` only after all three outputs pass validation. Keep source decoding and processing outside the web request that accepts the upload; acknowledge durable work, then let a bounded worker pool absorb the burst.

## Verify delivery before releasing a campaign

Verification has three layers. First, inspect the object itself: decode it, confirm JPEG, check dimensions and byte count, and reject a zero-length or truncated result. Second, render a seed message through representative clients and confirm that every signed link is reachable for the full campaign and expected read window. Third, observe the workflow: queue age, success ratio by policy version, processing duration, and the count of listings blocked on incomplete variants.

Do not confuse an image request with a human read. Hosted images can supply an open signal, but client-side proxying, prefetching, caching, and privacy controls weaken the inference. Treat opens as a noisy aggregate indicator, never as a reliable per-recipient fact or a dependency for business logic.

The release gate should be boring: every referenced manifest is `ready`, each template requests a named ratio that exists, and a seed delivery renders without attachment fallback. Sample visual output after any processor, codec, crop-model, or policy change. Smart crop deserves human review on a property corpus because a technically valid crop can still remove the front door, balcony, or room feature the campaign is meant to show.

## Roll back the policy, not the source

Make the manifest pointer the rollback lever. If `email-v2` produces unacceptable crops, stop promotion of that version and move campaigns back to the intact `email-v1` manifests while corrected jobs run. Do not delete the originals or overwrite known-good variants during rollout.

Roll out by a small listing cohort, compare validation failures and reviewed crops, then expand. Keep the previous version for at least the longest active email-link lifetime. If the processor is unavailable, pause new campaign eligibility and drain the durable queue after recovery; do not shift transformation into the synchronous send path under pressure. That preserves the sender's SLO and gives operations one bounded backlog to manage.

Rollback stays cheap.

## References

- MDN, “Image file type and format guide”: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- Cloudinary, “Image transformations”: https://cloudinary.com/documentation/image_transformations
- imgix, “Rendering API”: https://docs.imgix.com/apis/rendering
- ImageKit, “Image transformations”: https://imagekit.io/docs/image-transformation
- M3AAWG, “Email Tracking Protection Best Practices”: https://www.m3aawg.org/sites/default/files/m3aawg-email-tracking-protection-bp-2023-06.pdf
