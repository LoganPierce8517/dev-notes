# Multi-Tenant Help Center Retrieval: Citation Tracing Without Duplicating Every Index

Short answer: build retrieval around immutable article revisions and a tenant-scoped citation envelope, then page on the rate of answers whose cited revisions cannot be reproduced; don't use one physical index per tenant unless isolation requirements justify its multiplication of index and on-call cost.

The page fires at 02:13: `citation_replay_success` has fallen below its SLO. The on-call sees a customer-support answer, three displayed citations, the tenant identifier, the retrieval run identifier, and a red replay result: citation two points to article revision 184, but the trace does not contain the text span that supported the answer. Search latency is healthy. The bot still sounds convincing. This is a worse failure than an empty result because an agent may paste the answer into a customer reply before anyone notices that its evidence cannot be inspected.

Act on the trace, not the prose. Quarantine answers whose citation envelopes fail validation, preserve the retrieval inputs, and replay against the exact indexed revision. The earlier signal should have been a rising validation-failure ratio at answer assembly time, well before support agents produced enough traffic to burn the citation SLO.

No citation, no answer.

## How should retrieval for a multi-tenant SaaS help center preserve citation tracing?

Treat a citation as a verifiable data record rather than a URL appended after generation. Each retrieved chunk needs a tenant ID, stable article ID, immutable revision ID, chunk ID, content digest, and offsets into the normalized revision. The answer stores those fields alongside the retrieval run ID and the query policy version. A reader can then move in both directions: answer claim to chunk, and chunk to the exact help-center revision from which it came.

Tenant scope belongs in the retrieval predicate and in authorization, not merely in prompt text. A shared index can be economical when every record carries a mandatory tenant field and the query layer cannot omit that filter. A per-tenant index gives a stronger physical boundary, but its fixed overhead grows with tenant count even when most tenants have little content. A tiered design is often the sober choice: shared infrastructure for ordinary tenants, dedicated indexes for tenants whose contractual or regulatory isolation needs can pay for the operational boundary.

There is a catch. A shared index is not suitable when the search engine cannot enforce tenant filtering before ranking, because post-filtering the top results can return too few relevant chunks and makes isolation depend on application code. Stick with a dedicated index, or a search layer with enforceable pre-filtering, for that case. Conversely, don't create ten thousand tiny indexes merely because the data model has ten thousand tenant IDs; capacity planning has to include metadata, replicas, compaction, snapshots, and recovery time, not just vector bytes.

Citation tracing also changes update semantics. Never mutate revision 184 into revision 185 in place and leave old answers pointing at a moving target. Write the new revision, index its chunks, make the new revision active, and retain the old revision for the citation-retention window. Deletion is different: authorization may require the source text to become unavailable even though the audit record must retain identifiers and a digest. That policy boundary should be explicit, because reproducibility and deletion obligations can pull in opposite directions.

The original retrieval-augmented generation paper describes generation conditioned on retrieved non-parametric memory. For an internal knowledge-base bot, that retrieval boundary is where provenance must be captured; trying to reconstruct it from the final wording is guesswork. The model's answer is an output. The retrieval trace is evidence.

## Work backward from the page

The first dashboard should answer four questions without opening logs: which tenants are affected, which article revisions fail replay, whether retrieval or answer assembly lost the citation, and how much of the error budget is being consumed. A generic “RAG accuracy” gauge cannot do that. It merges relevance, grounding, authorization, source lifecycle, and formatting into one number, so the page arrives without an owner or an action.

Instrument the path as a sequence of counters and distributions. Count retrieval runs, empty candidate sets, tenant-filter rejections, assembled answers, citation envelopes, envelope-validation failures, and successful replays. Record search and assembly latency separately. Keep tenant IDs out of unconstrained metric labels if cardinality would overwhelm the telemetry system; expose affected tenants through sampled traces or a bounded diagnostic view instead. This is capacity planning applied to observability — telemetry can become its own indexing bill.

The SLO should describe what users can rely on: for example, the proportion of delivered answers for which every displayed citation is authorized, resolves to the recorded immutable revision, matches the stored digest, and contains the recorded span. The exact target cannot be borrowed from another team. I'm not sure where your acceptable curve bends without the support volume, answer risk, retention rules, and cost of suppressing a valid answer; those inputs resolve the uncertainty.

Use a stable failure taxonomy. `CITATION_SCOPE_MISMATCH` means the tenant in the answer context differs from the source record. `CITATION_REVISION_MISSING` means retention or ingestion state prevents replay. `CITATION_DIGEST_MISMATCH` means the bytes no longer match the recorded evidence. These are application-level classifications, not invented server failures, and each maps to a different owner. One bucket called `citation_error` will save a few lines of instrumentation and waste the on-call's first twenty minutes.

Here's a focused Go boundary for validating a retrieved citation before the answer is released. It deliberately takes the authorized tenant as an argument, so a caller cannot validate a record without making scope visible.

```go
package citation

import (
	"crypto/sha256"
	"errors"
)

var (
	ErrScopeMismatch = errors.New("CITATION_SCOPE_MISMATCH")
	ErrInvalidSpan   = errors.New("CITATION_INVALID_SPAN")
	ErrDigestMismatch = errors.New("CITATION_DIGEST_MISMATCH")
)

type Record struct {
	TenantID  string
	ArticleID string
	Revision  string
	ChunkID   string
	StartByte int
	EndByte   int
	Digest    [32]byte
}

func Validate(authorizedTenant string, revisionText []byte, r Record) error {
	if r.TenantID != authorizedTenant {
		return ErrScopeMismatch
	}
	if r.StartByte < 0 || r.EndByte < r.StartByte || r.EndByte > len(revisionText) {
		return ErrInvalidSpan
	}
	if sha256.Sum256(revisionText[r.StartByte:r.EndByte]) != r.Digest {
		return ErrDigestMismatch
	}
	return nil
}
```

This check is intentionally boring. It does not judge whether the passage is relevant or whether the generated claim follows from it; those need separate evaluation. It proves a smaller, operationally useful statement: the displayed citation is in scope and still identifies the same bytes selected during retrieval. Don't inflate that into “the answer is true.”

## Index cost follows copies, churn, and recovery

Vector count is only the first term. For tenant (t), a useful planning model is `chunks(t) × embedding_bytes × replica_count`, plus lexical structures, document fields, metadata, tombstones, and engine overhead. Add write amplification from article revisions and temporary overlap while old and new revisions coexist. Then price backup storage, restore bandwidth, and the engineer-hours required to test recovery. The cheapest steady-state index can be the expensive design during a tenant-wide reindex. Start with measurements from representative help centers: distribution of article sizes, chunk count after normalization, revision rate, query rate, and retention window. Use percentiles as well as averages because one documentation-heavy tenant can dominate storage and ingestion. Run a shadow reindex before committing to a shard topology. If a full rebuild takes longer than the recovery objective, either increase parallelism, reduce the rebuild unit, retain recoverable snapshots, or revise the objective honestly. Chunking policy belongs in this calculation because it is a cost control and a citation policy at the same time: smaller chunks increase record count and may detach a sentence from the heading or prerequisite that gives it meaning, while larger chunks reduce record count but can make citations vague and consume more of the model context. Keep normalized headings with their section text, set deterministic boundaries, and version the chunker. A change from `chunker_v3` to `chunker_v4` is an index migration, not a harmless parser refactor, because chunk IDs and offsets may change; during that migration, old citations still need their old normalization rules and revision bytes, new retrieval runs need the new policy version, and the capacity plan must carry both generations until the retention window permits removal.

Copies dominate.

Now compare operating models without pretending that one wins everywhere:

| Model | Index-cost shape | Isolation boundary | On-call burden | Lock-in pressure | Use it when |
|---|---|---|---|---|---|
| Shared managed index | Fixed cost is amortized; usage grows with total chunks and queries | Logical, dependent on enforced pre-filtering | Lower engine burden, but application policy remains yours | Query, filter, and export semantics may constrain migration | The team values reduced engine operations and can verify scope enforcement |
| Dedicated managed indexes | Per-index overhead grows with tenant count | Physical index boundary | Fleet provisioning and noisy configuration changes remain | Migration repeats across a larger fleet | A bounded tenant tier requires stronger isolation |
| Shared self-hosted index | Infrastructure is amortized; replicas and headroom are explicit | Logical, controlled by your query layer and engine | Highest direct responsibility for upgrades, compaction, backup, and restore | Lower service dependency, continued engine-format dependency | Search operations are a real team competency and utilization justifies it |
| Dedicated self-hosted indexes | Copies, shards, and idle headroom multiply | Strong physical boundary | Largest fleet and recovery surface | Architecture can still bind you to engine behavior | Isolation is mandatory and the organization funds the SLO |

The buy-vs-build decision is therefore an SLO allocation decision. Managed service can transfer engine maintenance, but it cannot transfer responsibility for tenant authorization, revision identity, citation semantics, evaluation data, or the suppression policy. Self-hosting grants control over placement and recovery, but every pager for compaction, disk watermarks, replica lag, and version upgrades lands somewhere on the platform roadmap. Put those pages in the cost model before comparing unit rates.

## Ship the trace before tuning ranking

Roll out in stages. First, make ingestion deterministic: the same tenant, revision, normalization policy, and chunker version should produce the same chunk identities and digests. Second, run retrieval in shadow mode and verify that every candidate carries complete provenance before any generated answer reaches a support agent. Third, enable answers for a low-risk slice while monitoring citation validation, empty retrieval, latency, and suppression. Ranking experiments can follow once the evidence path is dependable.

Build an evaluation set from approved help-center questions without mixing tenant corpora. Grade retrieval recall against eligible revisions, citation-span usefulness, and answer support as separate outcomes. A system can retrieve the right article and cite an unhelpful paragraph; it can also retrieve a useful paragraph and generate a claim that goes beyond it. One aggregate score hides both failure modes. Keep policy versions in the evaluation output so a score change can be traced to a chunker, embedding, filter, or reranker change.

The action attached to the page matters as much as the threshold. If validation fails, suppress or clearly withhold the affected answer, retain the trace under the applicable policy, and route the failure by taxonomy. If replay succeeds but relevance declines, roll back the ranking policy or stop its rollout. If one tenant drives ingestion lag after a bulk documentation import, rate-limit that workload without consuming every other tenant's freshness budget. These are different incidents even if the support UI merely shows “no answer.”

Be careful with sensitivity. A threshold that pages on one failed citation can wake someone for a single stale test record; a threshold averaged across the whole fleet can hide a total failure for a small tenant. Use a minimum event count with a burn-rate view, and pair the fleet SLO with a bounded tenant-impact detector. The settings must be tested against real traffic distributions, not chosen because 99.9% looks serious.

False positives have a measurable cost: interrupted sleep, desensitized responders, and pressure to disable the detector. False negatives spend customer trust. The right final step is an alert review after each firing — was there a user-visible, actionable citation failure, and did the alert arrive early enough to change the outcome? If the answer is no, adjust the signal or routing before raising the threshold by reflex.

## References

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401

## Further reading

- https://arxiv.org/abs/2005.11401
