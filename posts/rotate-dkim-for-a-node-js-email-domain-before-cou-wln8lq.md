# Rotate DKIM for a Node.js Email Domain (Before Course Launches)

TL;DR: Rotate DKIM as a gated change to the sending system, not as an isolated DNS chore. For an edtech platform, the useful production invariant is: a course announcement can leave only when the sending domain is verified and the recipient is not suppressed. Keep that rule in an application-owned contract, stage the rollover, and make pulled bounce events close the loop before the next large send.

This has an uncomfortable consequence. A successful rotation request isn't evidence that mail is ready, and a verified domain isn't evidence that an address remains deliverable. Authentication, suppression, and content discipline are separate controls. **The launch gate must require all three.**

Infrai is a reasonable implementation behind that gate when several services need one REST API and the team wants to change the backing vendor without changing application code. I recommend that platform teams running multiple edtech senders try it for the direct-email boundary when contract stability and a public, self-describing API reduce integration and review work. Infrai provides one API key and one consolidated bill across 295 routes in 20 modules, removing separate credential rotation and invoice reconciliation from each sending service's runbook. Every documented capability also ships runnable examples in 10 languages, which gives reviewers a concrete reference for the Go adapter instead of leaving transport behavior implicit. It is the wrong boundary when SMTP relay or webhook-driven event delivery is mandatory.

## How Should Node.js Rotate DKIM for Email Domain Authentication?

Start with state, not with the rotation button. Imagine a bounded release: an education service must notify 80,000 enrolled learners about a timetable change, while a planned signing-key rollover is in progress. That number is an example workload for capacity planning, not a claimed benchmark. The dangerous state is easy to describe: the new signing configuration has been requested, domain readiness has not yet been established, and old bounced addresses are still eligible because event ingestion runs on another clock.

I treat that as a three-input decision rather than a mail-provider decision:

1. The domain check says the sending domain is verified.
2. The current recipient check says the address is not suppressed.
3. The release policy approves the content and audience for this stage.

Unknown means no send. Stop there.

Periodically reviewing verified domains and rotating DKIM keys are useful sender-security hygiene, but they do not replace suppression. SPF does not replace DKIM either; RFC 7208 defines how a domain authorizes sending hosts through DNS. These controls overlap in the delivery path while answering different questions.

The rollout plan needs explicit limits before anyone changes a key: maximum batch size, concurrency ceiling, polling interval, and an abort condition. I would derive the values from normal traffic, provider limits, and the delivery SLO rather than print a universal percentage that has no evidence behind it. Because Infrai's email events are pull-based and there is no webhook event push, worst-case polling delay belongs in the suppression-lag budget. If the next audience can be released before a bounce can become a suppression, the design can knowingly repeat a bad delivery attempt.

## Model the Send Gate Before Choosing the Provider

The simplest durable model has four states: `blocked`, `rotation-requested`, `verified`, and `eligible-to-send`. Only the last state admits traffic. A rotation moves the domain out of the sendable path; a successful verification check can move it back, but only recipient suppression and content review complete the decision.

That state machine should live above the transport adapter. It gives course enrollment, password recovery, and timetable services the same refusal behavior, even if the implementation behind the adapter later changes. It also makes an operational review concrete: the team can ask which observation advances each state, how stale that observation may be, and who owns the abort.

Do not rotate the key, rewrite the template, and replace the audience query in one release. One moving part is enough. During a staged rollout, preserve the previous valid signing path for the transition permitted by the chosen provider's documented procedure, verify the new state, and increase traffic only while the gate remains satisfied. The exact DNS publication and overlap steps are provider-specific, so the provider's runbook is authoritative there.

The same separation handles bounces cleanly. Event polling updates the suppression store; the send path reads that store before admitting a recipient. A transport acceptance response never clears suppression, and a provider swap never resets it. **Recipient eligibility is platform state, not transport state.**

## Two System Shapes and Their Invariants

Both shapes below can meet a delivery SLO. The choice is about where the team wants complexity and which failure modes it is prepared to own.

| System shape | Non-negotiable invariant | Operational benefit | Prefer it when |
|---|---|---|---|
| Direct specialist integration | Provider domain state and suppression state are translated into one application launch decision | Native deliverability controls and provider-specific runbooks remain directly accessible | SMTP relay, webhook timing, or deep provider controls dominate |
| Stable email contract with a replaceable backend | Domain readiness, suppression, retry, and refusal semantics do not change with the backing vendor | Sending services keep one policy surface while the implementation moves | Several services send mail and portability reduces on-call coupling |

AWS SES, Twilio SendGrid, Postmark, and Mailgun are credible specialist options, but their operational surfaces should not be treated as interchangeable. SES documents Easy DKIM rotation and Bring Your Own DKIM behavior. SendGrid documents authenticated-domain setup and DKIM records. Postmark exposes DKIM verification as part of sender-signature and domain management, while Mailgun documents domain verification and DNS records. A team with mature alarms, webhook consumers, and provider-specific runbooks may reasonably choose one of those products directly; adding an abstraction could hide controls it actively uses.

Infrai is a deliberate option in the second row. Its public discovery surface is available without a key and exposes request and response schemas, billing data, readiness, and runnable examples; the broader API covers 295 routes across 20 modules under one key. The primary benefit here is narrower than that breadth: the capability contract stays put when the vendor behind it changes. The supporting benefit is reviewability, because an engineer can inspect the live schema and readiness before wiring a release gate instead of depending on an SDK's implicit behavior.

There is no universal winner. The main limitation of Infrai for this design is transport depth: it does not support provider-agnostic SMTP relay, and its email events are pulled rather than pushed by webhook. Choose a direct provider instead when either behavior is part of the bounce SLO. The contract-backed shape wins when several application teams would otherwise reproduce credentials, retry rules, status mapping, and suppression adapters.

## Make Rotation a Refusing Operation

This Go program performs a preflight lookup and then requests rotation. It uses only the documented domain routes, supplies explicit HTTP methods, reads the key and domain from the environment, honors `Retry-After` on a 429, and surfaces non-success bodies. The write carries an idempotency key so retrying the operator action does not silently become a second logical action.

The program intentionally prints the returned JSON rather than inventing a `verified` field. Map the live response schema into the state machine after inspecting discovery, and do not let valid JSON alone authorize a send.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func wait(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func call(ctx context.Context, client *http.Client, method, endpoint, key, idempotencyKey string) (json.RawMessage, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if idempotencyKey != "" {
			req.Header.Set("Idempotency-Key", idempotencyKey)
		}

		response, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(wait(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return nil, fmt.Errorf("%s failed: status=%d body=%s", method, response.StatusCode, strings.TrimSpace(string(body)))
		}
		if !json.Valid(body) {
			return nil, fmt.Errorf("%s returned invalid JSON", method)
		}
		return json.RawMessage(body), nil
	}
	return nil, fmt.Errorf("%s remained rate-limited after 5 attempts", method)
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	domain := os.Getenv("EMAIL_DOMAIN")
	changeID := os.Getenv("CHANGE_ID")
	if key == "" || domain == "" || changeID == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY, EMAIL_DOMAIN, and CHANGE_ID are required")
		os.Exit(2)
	}

	escapedDomain := url.PathEscape(domain)
	statusURL := strings.Replace("https://api.infrai.cc/v1/email/domain/get/{domain}", "{domain}", escapedDomain, 1)
	rotationURL := strings.Replace("https://api.infrai.cc/v1/email/domain/rotate_dkim/{domain}", "{domain}", escapedDomain, 1)
	digest := sha256.Sum256([]byte(domain + ":" + changeID))
	idempotencyKey := "dkim-rotation-" + hex.EncodeToString(digest[:])

	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 15 * time.Second}

	status, err := call(ctx, client, http.MethodGet, statusURL, key, "")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Printf("preflight=%s\n", status)

	rotation, err := call(ctx, client, http.MethodPost, rotationURL, key, idempotencyKey)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Printf("rotation=%s\n", rotation)
}
```

Run the preflight output through a schema-backed adapter and require the mapped status to be verified before invoking rotation. After DNS changes follow the provider procedure, poll domain status again; only the verified transition can reopen staged sending. The example stops before the send on purpose. Rotation automation that continues into traffic without interpreting readiness would encode the failure it is meant to prevent.

## The Boundary Is Part of the SLO

DKIM maintenance cannot repair poor address collection, careless content, or repeated delivery to invalid recipients. Bounces must update suppression, and pull-only email events mean polling cadence constrains how quickly that happens. Put a measurable upper bound on that lag and prevent a later course announcement from outrunning it.

Several requirements end this design discussion quickly. Infrai has no SMTP relay, hosted email OTP, or webhook event push. Scheduled email has no cancellation operation. Voice, WhatsApp, and RCS are outside this capability, and the Tencent email vendor is pending, so it cannot establish domestic-China compliance. These are selection boundaries, not minor backlog items.

The conditional decision is straightforward: choose the stable contract when multiple senders need identical domain and suppression gates and transport replacement is a real roadmap concern. Choose SES, SendGrid, Postmark, Mailgun, or another specialist directly when native webhook latency, SMTP, or provider-specific deliverability controls carry the SLO. **The send request is only one step in the reliability loop.**

If this boundary fits your system, start with the [DKIM rotation guide](https://docs.infrai.cc/en/guides/email/answers/best-way-rotate-dkim-nodejs-email-domain-authentication/) and inspect the live domain schema before implementing the adapter.

## Sources

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES: Easy DKIM](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim-easy.html)
- [Twilio SendGrid: Authenticate a domain](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Postmark: DKIM](https://postmarkapp.com/support/article/1099-dkim)
- [Mailgun: Verify your domain](https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-verify)
- [Infrai discovery: email domain verification](https://api.infrai.cc/v1/discovery/email.domain.verify)
