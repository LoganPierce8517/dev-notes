# Support Ticket Tagging: An Alternative to Fine-Tuning for Portable CRM Actions

The page that wakes an on-call engineer is rarely the first failure in an AI tagging pipeline. For support ticket tagging, a practical alternative to fine-tuning is to compare an embedding classifier, zero-shot inference, and reranking against the same evidence contract. In this customer-support workflow, sales-call summaries become CRM actions, so the useful alert is a rising rate of uncertain actions: a proposed renewal task, owner, or deadline that cannot be supported by transcript and policy evidence. That signal lets the team inspect the decision boundary before bad tags become workflow state.

**TL;DR:** Treat fine-tuning as a later optimization, not the first answer. Start with a portable contract and a replayable evaluation set, then choose an embedding classifier, a zero-shot language-model path, or a reranker according to label stability, abstention needs, and on-call capacity. Alert on unsupported CRM actions and calibration drift, not on model choice.

The portability question is whether the same labeled examples, thresholds, evidence fields, and audit records can move with the application when the inference provider changes.

## What should wake the on-call engineer?

Imagine the alert page at 09:10: the action-support rate has crossed its threshold for three consecutive windows, while transcript ingestion and request latency remain within their service-level objectives. The on-call view shows a sample of proposed CRM actions, the label distribution, abstention rate, and the evidence spans used to justify each action.

Work backwards. A transcript summary is produced, candidate labels are generated, a decision is made, and only then is a CRM mutation allowed. The earlier signal is disagreement with a small reference set tagged by humans with the action, owner, due-date policy, and an explicit no-action outcome. A second signal is evidence coverage. If a task says a prospect requested a follow-up but no supporting span survived normalization, the pipeline should abstain or route to review.

This is an operations problem before it is a model problem. A high-confidence wrong action is more dangerous than visible abstention because downstream staff will trust the resulting state. The instrumentation change is a provider-neutral decision record at every boundary:

\`\`\`go
type DecisionRecord struct {
	TraceID       string
	Label         string
	Score         float64
	Margin        float64
	Evidence      []string
	PolicyVersion string
	ModelFamily   string
	ProviderRef   string
	Outcome       string
}
\`\`\`

\`ProviderRef\` is audit metadata, not a dependency in the decision format. The classifier can accept text and return candidates, scores, and evidence references while the application keeps its decision schema. That makes a migration a controlled replay instead of a rewrite.

Set separate thresholds for descriptive tags and workflow-changing actions. One global cutoff is convenient, and usually wrong. If a false positive creates an externally visible CRM task, its reversal cost is higher than the cost of asking a human to label an ambiguous call.

The page changes.

## Can support ticket tagging use an alternative to fine-tuning?

An embedding classifier asks whether a new summary is close to known labeled examples. A zero-shot language-model path asks the model to apply a label schema without task-specific training. A reranker orders a supplied set of candidate labels or actions using the text and their definitions.

An embedding classifier fits a slowly changing taxonomy with credible examples and a need for fast local scoring. Store vectors with label, policy version, and tenant scope; retrieve a bounded candidate set; then apply a calibrated classifier or threshold. A relational vector extension can keep metadata and similarity search together, but similarity is evidence for a decision, not a calibrated probability. The trade-off is operational: this path needs representative labels and a calibration review, while zero-shot inference trades that preparation for more variability at runtime.

Zero-shot inference helps when definitions exist but examples are sparse. Its weak point is repeatability: prompt wording, context length, structured-output handling, and provider behavior become contract inputs. Validate the returned label against an allowlist, require an abstention value, and retain the schema and policy version.

A reranker fits a known candidate set when lexical matching is brittle. It does not solve candidate generation. If the correct action never enters the candidate set, a perfect ranking stage cannot recover it. Count that failure separately.

For sales-call actions, the boundary is mixed: stable categories such as \`pricing_question\` can tolerate an embedding first pass, while \`create_follow_up\` needs evidence and policy checks. A single pipeline may use different methods by risk class if its output contract stays identical.

## How do you compare them without trapping the application?

Build the comparison around replay, not a live traffic experiment. Include ordinary calls, short calls, several-speaker calls, poor transcripts if they occur in the workload, and explicit no-action cases. Keep the human label, evidence span, policy version, and allowed action separate from the text. A model-generated summary must not become the only ground truth.

| Dimension | Embedding classifier | Zero-shot language model | Reranker |
| --- | --- | --- | --- |
| Needs labeled examples | Usually yes | Few to start | Candidate examples or definitions |
| Main failure | Similar wording, wrong intent | Inconsistent interpretation | Correct label absent from candidates |
| Strong control | Local threshold and retrieval set | Schema validation and abstention | Candidate-set ownership |
| Portability asset | Vectors, labels, calibrator | Prompts, schemas, replay set | Candidates, definitions, ranking logs |
| Best operational question | Is similarity calibrated? | Is output repeatable? | Did generation miss the answer? |

Do not reduce the result to accuracy. Track per-label recall, false-positive rate for workflow actions, abstention rate, evidence coverage, p95 latency, review queue age, and the fraction of decisions replayable from retained inputs. A macro average can hide a rare but consequential label such as \`legal_review_required\`.

Capacity planning belongs in the same worksheet. Let \`N\` be calls per hour, \`p\` the proportion routed to a path, and \`r\` sustained decisions per worker at the latency target. Rough concurrency is \`N * p / r\`; add burst and retry headroom, then check review capacity. Cheap inference does not help if the review queue grows faster than the team can clear it.

The portability test is simple: can a second implementation consume the replay set and emit the same decision schema without importing the first SDK's types? If not, the business rule is coupled to an inference client. Keep adapters at the edge and make provider-specific request fields disposable.

## Where should the alert threshold sit?

A tag may be written automatically only when the classifier passes the label threshold, evidence check, and policy check. Otherwise it becomes a review item. Express policy as data so an owner-rule change does not require an inference-adapter change.

\`\`\`go
type ActionDecision struct {
	Action    string
	Score     float64
	Evidence  []string
	Approved  bool
	Reason    string
}

func Decide(score float64, evidence []string, threshold float64) ActionDecision {
	if score < threshold {
		return ActionDecision{Approved: false, Reason: "below_threshold"}
	}
	if len(evidence) == 0 {
		return ActionDecision{Approved: false, Reason: "missing_evidence"}
	}
	return ActionDecision{Score: score, Evidence: evidence, Approved: true}
}
\`\`\`

The code is intentionally boring. The threshold comes from policy, evidence is explicit, and the caller can record a rejection without calling it a model failure. The production policy should also bind the action to an owner and permitted CRM operation.

Alert on sustained change, not one strange call: unsupported approved actions, falling evidence coverage, a shifted label distribution, or increased replay disagreement after a provider or prompt change. Include a redacted sample and policy version. An alert that cannot be investigated is a notification tax.

False positives have a cost too. A threshold that sends half the calls to review may preserve precision while violating the human-review SLO. A threshold that approves nearly everything looks efficient until an owner spends a morning reversing tasks. Tune the operating point per risk class, record the reason, and review it against fresh adjudicated samples.

The practical trap is a clean dashboard with an unexamined queue behind it: a provider change can preserve aggregate accuracy while moving borderline calls from one label to another, and a prompt change can preserve the approval rate while removing the evidence spans that made an action reviewable. Keep a fixed replay slice for every policy version, sample both approved and abstained decisions, and compare the queue's age with the review SLO before raising any threshold. If the evidence field is empty, the reviewer should see that absence as a reason to reject the action, not as a missing display field. This is slower than treating the model score as the whole signal, but it gives the platform team a decision they can defend when CRM state has to be corrected.

## The decision rule

Use an embedding classifier when labels are stable, examples are credible, and retrieval plus calibration can meet latency and review SLOs. Use zero-shot inference for taxonomy discovery or sparse-label bootstrapping, with strict schemas and abstention. Add reranking when candidate generation is reliable but plausible actions need semantic ordering. These stages can compose.

The durable choice preserves the evidence trail and keeps the action contract portable. Fine-tuning may improve a mature, high-volume boundary, but it does not replace label governance, replay data, calibration, or an alert for unsupported CRM mutations. The limitation is unavoidable: a portable interface cannot make a sparse or contradictory taxonomy accurate. Those controls are what the on-call engineer can operate.

## Further reading

- OWASP, *Top 10 for Large Language Model Applications*: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- pgvector, *Open-source vector similarity search for Postgres*: https://github.com/pgvector/pgvector/
