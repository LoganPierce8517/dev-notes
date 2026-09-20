# How to Check Asynchronous Generated Video Job Costs and Capabilities

The page says promotional clips are missing from the e-commerce media library. On-call sees a successful generation request, a pending job, and no searchable tags; retrying risks creating another clip, while caching the empty search result can hide a clip after it is ready. Short answer: generated video is asynchronous because accepting work and finishing an encoded, stored, searchable asset are distinct events. Track those events under one job ID, check output capabilities before submission, and alert on the age of work that has stopped progressing, not on the duration of one HTTP request.

## Why does generated video need an asynchronous job model?

Work backwards from the missing search result. The request was accepted, generation ran, the output was stored, tagging read the output, and the index published the tags. These are separate transitions, and a green HTTP response at the first transition says nothing about the last one. HTTP 202 explicitly means a request has been accepted for processing but processing has not completed. It does not promise eventual success.

The earliest useful signal is the age of the oldest accepted job whose next required transition has not arrived, partitioned by stage. An alert on end-to-end search latency alone arrives after the storefront team has already noticed the gap. A count of accepted requests is worse: it can look healthy while the storage writer is stuck. Record accepted, generating, stored, tagged, indexed, failed, and canceled transitions with a stable job ID, a bounded stage label, and timestamps. Do not attach raw prompts or asset IDs as metric labels; their cardinality grows with every clip. Keep IDs in logs or traces for investigation.

This also answers the cost question more honestly than a per-generation price comparison. A completed clip may incur object storage, derived thumbnail storage, index entries, and cache occupancy; repeated submissions multiply that footprint, while a short cache lifetime can increase origin reads. Measure bytes retained and requests served from each tier before claiming one pipeline is cheaper. No universal break-even number follows from the job model. Consider the particular failure where two requests for the same promotion generate distinct outputs while the tagger is paused: both outputs consume retention space, neither appears in search, and a cache of the empty query response delays visibility even after indexing resumes. Separate the generation bill from bytes actually retained and bytes read again by tagging, indexing, and viewers; otherwise the cost report can improve while the library becomes less useful.

That gap matters.

## How should a job move toward searchable output?

Use a durable job record with separate generation and publication state. The worker must not mark the job searchable until the output exists, the tagger has read the intended version, and the index acknowledges its update. If a worker crashes after storing a clip but before advancing state, retry that stage against the same job ID and output key; avoid resubmitting generation as the default recovery path. Terminal failure should retain the failed stage and an actionable error category. Cancellation is a request, not proof that already-produced bytes vanished.

The following small Go program models the stage gate. Its timestamps are illustrative test inputs, not measured service latency; the failure path is deliberately explicit.

```go
package main

import (
    "fmt"
    "time"
)

type Stage string

const (
    accepted Stage = "accepted"
    generating Stage = "generating"
    stored Stage = "stored"
    tagged Stage = "tagged"
    indexed Stage = "indexed"
)

type Job struct {
    ID string
    Stage Stage
    ChangedAt time.Time
}

func advance(j *Job, next Stage, at time.Time) error {
    allowed := map[Stage]Stage{accepted: generating, generating: stored, stored: tagged, tagged: indexed}
    if allowed[j.Stage] != next || at.Before(j.ChangedAt) {
        return fmt.Errorf("invalid transition %s to %s", j.Stage, next)
    }
    j.Stage, j.ChangedAt = next, at
    return nil
}

func main() {
    start := time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC)
    job := Job{ID: "promo-42", Stage: accepted, ChangedAt: start}
    for i, stage := range []Stage{generating, stored, tagged, indexed} {
        if err := advance(&job, stage, start.Add(time.Duration(i+1)*time.Minute)); err != nil {
            panic(err)
        }
    }
    fmt.Printf("%s searchable=%t\n", job.ID, job.Stage == indexed)
}
```

A production record needs atomic transitions and idempotent side effects; this in-memory example provides neither. That is its limitation. Persist the expected prior stage with each update, deduplicate submission by a caller-provided idempotency key, and make the index operation safe to repeat. Test worker termination between storage and tagging, a duplicate completion event, and a stale index update before relying on the alert. The choice of queue matters less than proving those three cases.

## Which capabilities belong in the preflight check?

Before a promotional clip enters the queue, check that the requested duration, aspect ratio, input assets, output container, and delivery method fit the selected generation backend's published contract. Then check the downstream reader: a video output that the tagging worker cannot decode is not a usable library asset. The image-format guide in Further reading is relevant to thumbnails and image inputs, not evidence that any service can generate a particular video format. Store the actual content type with each asset and validate it at the boundary.

A capability check should return a reason that the requester can act on, and it should run again in the worker if capabilities can change between submission and execution. Do not infer support from a successful request-acceptance response. Keep a small fixture set containing a vertical clip request, an unsupported input type, and an output that can be stored but not indexed. The last case catches a misleading success metric: generated is not searchable.

Check both ends.

## Where does the alert threshold earn its keep?

Start with an SLO for accepted-to-searchable time and measure its distribution by stage; choose a page threshold only after observing the normal tail and the business deadline for a promotion. Capacity planning then follows the slowest stage: arrival rate, concurrent generation slots, output bytes per accepted job, retention, tagging throughput, index lag, and cache hit rate. If the index lags while generation is healthy, more generation capacity increases the backlog and storage bill.

| Approach | Operational control | Cost exposure |
| --- | --- | --- |
| Managed generation, owned publication pipeline | Less generation scheduling to run; tagging and indexing remain on-call responsibilities | Output retention, duplicate work, and cache misses still need accounting |
| Self-hosted generation and publication | Direct control of worker placement and data lifetime | Idle capacity, failed retries, storage, and cache operations remain internal |

Neither row settles a purchase decision. The limitation of managed generation is reduced control over scheduling and output retention; self-hosting instead transfers capacity and failure recovery to the platform team. Collect job age by stage and retained bytes per usable searchable clip in a representative workload, then weigh lock-in against the team's on-call budget. A low threshold pages on ordinary queueing and teaches responders to ignore the page; a high threshold makes the first real signal the missing promotion itself. Count false pages alongside missed deadlines when tuning it.

## References

- https://www.rfc-editor.org/rfc/rfc9110.html#section-15.3.3
- https://www.rfc-editor.org/rfc/rfc9111.html
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types

## Further reading

- https://www.rfc-editor.org/rfc/rfc9110.html#section-15.3.3
- https://www.rfc-editor.org/rfc/rfc9111.html
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
