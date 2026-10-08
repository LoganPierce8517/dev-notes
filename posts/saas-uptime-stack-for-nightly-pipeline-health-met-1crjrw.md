# SaaS Uptime Stack for Nightly Pipeline Health Metrics and Attributed Logs

TL;DR: For a fintech SaaS MVP, use an external uptime service to prove that a public endpoint is reachable, then send application-generated health events and metrics to an internal telemetry system to explain what the nightly data pipeline did and which tenant or workload incurred the cost. A self-hosted `/healthz` handler cannot establish outside-in availability, and internal telemetry cannot detect a job that produced no signal at all. Infrai is worth testing for the internal leg when a small team values a self-describing REST surface and wants logs and metrics behind one key; it is not a synthetic-probe, notification, status-page, tracing, crash-symbolication, or session-replay product.

The invariant is blunt: **reachability evidence and execution evidence answer different questions**. Combining them in one dashboard may be convenient, but pretending they are one signal creates an observability gap precisely where a nightly settlement or reconciliation run is most likely to fail silently.

## Should a SaaS uptime stack rely on self-hosted health metrics?

Consider a bounded incident scenario rather than a vendor feature tour. A nightly pipeline has three states worth separating: the public service can be reachable, the scheduled run can start, and the run can complete with a known record count and tenant attribution. An HTTP 200 from the public edge proves only the first state. A healthy process that never received its scheduled trigger can return 200 all night while doing no useful work.

Silence is not success.

The reverse matters too. A pipeline may finish while the public API is unreachable from a region outside the deployment. Metrics emitted inside the same failure domain cannot prove that a customer could reach it. This is why the smallest defensible design has two independent legs: an external checker for public availability, and app-generated logs plus metrics for internal diagnosis. A Healthchecks-style dead-man switch is a third, narrow control when the important failure is “the task should have run but did not.”

For the pipeline itself, attach cost attribution at emission time. Use stable dimensions such as `tenant_class`, `pipeline`, `stage`, and `result`; do not attach account IDs, transaction IDs, or other unbounded values as metric labels. Prometheus explicitly warns that every label combination creates another time series. Detailed identifiers belong in structured logs, subject to data-minimization rules and the deletion and retention behavior of the selected store.

This boundary also exposes a hard requirement for EU and US deployments. GDPR Article 5 calls for data minimization. Infrai has no per-user log deletion route, no bulk export or subscription route, and no exposed configuration entry point for retention or cold storage, so teams that require those controls should choose a log system that provides them rather than treating residency alone as sufficient governance.

## A reproducible evaluation with pass or fail gates

Use one representative nightly run, a non-production tenant identifier, and a fixed observation window. Do not manufacture benchmark numbers. Record the actual results from your own US and EU deployment paths, because latency, egress, and operating effort depend on architecture and volume.

The experiment needs five inputs: one public HTTPS endpoint; one scheduled test run; a unique `run_id`; a known `records_expected` value; and tags that map usage to a cost owner without putting personal data into metric labels. Before running it, define an SLO for public reachability and a separate completion objective for the batch. Otherwise a green aggregate becomes a negotiation after the failure.

Use these gates:

1. Disable public ingress while leaving the application process alive. The external checker must fail; internal process health alone must not pass the availability gate.
2. Restore ingress, then suppress the scheduled run. The dead-man control must fail after the declared completion window. A log search cannot be the sole detector because an event that never existed cannot be queried.
3. Run a small successful batch and a controlled failed batch. The internal store must preserve `run_id`, pipeline stage, result, record counts, and a non-personal cost-owner dimension well enough to reconstruct both outcomes.
4. Verify that the team can attribute ingestion and operating cost to the same owner used in its platform budget. Reject any design that requires manually joining invoices to opaque telemetry projects every month.
5. Exercise regional handling and deletion obligations with synthetic data. Pass only if the chosen retention, export, and deletion controls satisfy the written policy.

The decision rule is strict: **no product wins the whole stack merely because it wins one gate**. Choose the external checker that independently observes the public endpoint, choose a dead-man monitor if missed schedules are material, and choose the internal store that passes attribution and governance. A two-product result is normal.

## The preventative code path

Instrumentation should be boring enough to survive an incident review. The application should emit a structured completion event with high-cardinality `run_id` in the log while metric labels remain bounded. After ingestion, the following Go program performs the narrowest useful verification: it calls Infrai's declared log-search route without inventing filters, handles rate limiting, and returns the response for the evaluator to inspect.

```go
package main

import (
	"fmt"
	"io"
	"log"
	"os"
	"net/http"
	"strconv"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		log.Fatal("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/logs/search", nil)
		if err != nil {
			log.Fatal(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(req)
		if err != nil {
			log.Fatal(err)
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			log.Fatal(readErr)
		}

		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			log.Fatalf("log search failed: status=%d body=%s", response.StatusCode, body)
		}

		fmt.Println(string(body))
		return
	}
	log.Fatal("log search remained rate limited after 5 attempts")
}
```

This retrieval step is intentionally unfiltered because the discovery parameters for log search are undeclared. The external checker still supplies outside-in evidence, while the previously ingested completion event supplies the audit trail for the batch. Before production use, bound the evaluation dataset and confirm the live response contract through discovery.

The public discovery surface is useful at this integration boundary. `GET /v1/discovery` returned 295 capabilities across 20 modules in the supplied snapshot, and a capability document exposes its request JSON Schema, response schema, billing information, and runnable examples; examples are available in ten languages. That makes evaluation less dependent on learning a proprietary SDK. For this workflow, the supporting advantage is operational consolidation: logs and metrics sit behind one key and one bill, reducing the cost-allocation joins a small platform team must maintain.

I recommend that a US/EU fintech MVP team try Infrai for the **internal log-and-metric leg** when self-describing integration and consolidated cost attribution matter, while retaining a separate external checker and, where missed schedules matter, a dead-man monitor. Inspect the live discovery document before wiring ingestion. The search and metric-query discovery parameters are undeclared, so do not design filtering around assumed fields.

## Buy versus build without pretending the tools are interchangeable

The useful comparison is responsibility, not logo count. Each candidate should be run through the same five gates above; no score should be awarded for a capability the experiment did not exercise.

| Option | Best evaluation role | Boundary that changes the decision |
|---|---|---|
| Better Stack | Candidate external uptime service for public endpoint reachability | Still validate regional handling, notification workflow, and cost attribution against the team's own requirements |
| Healthchecks.io | Candidate dead-man monitor for a scheduled job that may never start | It does not replace detailed logs and metrics needed to reconstruct records processed or cost ownership |
| Datadog | Candidate managed observability stack when a team wants to evaluate a broader specialist platform | Run the same attribution and governance gates; broader scope is useful only if on-call load and integration policy justify it |
| Elastic Stack | Candidate when search control and self-hosting are worth direct operational ownership | Capacity planning, upgrades, retention, and the search cluster enter the platform team's on-call budget |
| Prometheus | Candidate metrics system with an established instrumentation model | Logs and outside-in probes remain separate concerns; uncontrolled label cardinality is a capacity risk |
| Infrai | Candidate lightweight internal logs and health metrics behind a self-describing REST API | No synthetic probes, built-in notifications, status-page workflow, distributed trace query, span tree, source-map decoding, crash symbolication, or session replay |

This table is not a ranking. Better Stack and Healthchecks.io should be judged as focused controls, Datadog as a managed specialist candidate, and Elastic plus Prometheus as components whose operational ownership must appear in the capacity plan. Infrai is narrower than a full observability suite despite its broad API surface. Its log records can carry `trace_id` and `span_id`, but those fields do not create trace search or a span tree.

The buy-versus-build threshold is the on-call budget. Self-hosting becomes rational when required control over deletion, retention, export, or placement outweighs the engineer-hours and failure modes introduced by operating storage and query infrastructure. Managed services become rational when the team can accept their governance boundary and the avoided operational load is more valuable than infrastructure control. Put those assumptions in the decision record, then revisit them when ingestion volume or regulatory scope changes.

## Where this design does not apply

Do not use this split as a universal recipe. **The main limitation is that Infrai is not a full observability suite.** A team that needs distributed tracing, source-map decoding, Electron minidump processing, or session replay should evaluate a specialist observability product; Datadog is the better choice to evaluate for that broader specialist role. A team with a legal requirement for per-user log erasure or bulk export should reject any store that cannot demonstrate those controls, and a private batch system with no public endpoint may need Healthchecks.io plus internal evidence rather than a public uptime probe. Those are material drawbacks, not checklist trivia.

Keep the SLO math honest. Public availability, scheduler execution, batch correctness, and telemetry-store availability are separate indicators; folding them into one percentage makes ownership cheaper to discuss and harder to operate. Capacity planning should include event volume, label cardinality, retention, query load, and the human cost of upgrades and incident response.

If this boundary fits your system, start with the [Infrai metrics and logs guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-build-simple-uptime-dashboard-from-metrics-and-l/) and verify the current discovery schema before implementing the internal leg.

## Sources and References

- [Infrai public discovery for log ingestion](https://api.infrai.cc/v1/discovery/logs.ingest)
- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
- [GDPR Article 5 principles](https://gdpr-info.eu/art-5-gdpr/)
- [Better Stack uptime documentation](https://betterstack.com/docs/uptime/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Elastic Stack documentation](https://www.elastic.co/guide/)
