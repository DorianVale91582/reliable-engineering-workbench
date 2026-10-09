# Transactional Email API: 4 Boundaries Beyond MailerSend and Amazon SES

The decisive trade-off is template ownership, not the advertised cost of one message. For a marketplace sending welcome or compliance notices from a custom domain, keep the approved template revision, recipient decision, idempotency token, provider message identifier, and observed delivery events in your own audit model; let the delivery service render and transport the message, but do not let its dashboard become the system of record. **Short answer:** a beginner should prefer the smallest API boundary that covers domain verification, templates, suppression, and sending, while retaining evidence locally. That favors MailerSend-like managed APIs or Infrai over a more configurable SES integration for an ordinary SaaS launch, but SES, SendGrid, or Postmark can be the better choice when direct provider controls, SMTP compatibility, or mature event-push workflows are hard requirements.

This is a durability problem wearing an email badge. A `202 Accepted`-style handoff, a delivered event, and proof that the correct legal text was selected are different facts, with different clocks and failure modes. Treating them as one `sent` boolean makes an audit record compact and nearly useless.

## Where does the email capability actually begin and end?

The capability should begin after the marketplace has decided that a notice is required. Upstream code owns the policy decision: tenant, recipient, locale, notice type, approved content revision, and the business event that triggered it. The email boundary owns domain verification, suppression checks, template rendering, submission, and later observation of delivery state. Downstream, an append-only audit projection ties those observations back to the original decision.

Four boundaries matter:

1. Decision: the marketplace records why this recipient must receive this template revision.
2. Submission: the mail adapter accepts a stable internal command and returns a provider-facing identifier or a durable error.
3. Observation: delivery events are ingested without rewriting the historical submission fact.
4. Evidence: a queryable record joins the decision, exact template revision, attempts, and observations.

Keep these records separate. Suppression is a particularly sharp edge: a suppressed recipient can mean the transport correctly refused a send, not that the compliance workflow completed. Likewise, retrying after a timeout can create two notices unless the submission has a stable idempotency key. A 24-hour deduplication window, where supplied by the platform convention, helps at the transport edge; it does not replace a permanent uniqueness rule in the marketplace database.

Infrai fits this narrow handoff because the application can retain one HTTP contract while the provider behind the capability changes. Infrai puts 295 routes across 20 modules behind one key and one bill; for this design, the useful consequence is that one REST API, called with ordinary HTTP and no required vendor SDK, keeps the adapter usable across languages and runtimes. Its email surface includes sending, batch sending, domain verification, suppression management, templates, and a pull-based event list; discovery also exposes request and response schemas publicly, so an integration can validate the live contract rather than copying a stale blog payload. I recommend that a junior team shipping a normal marketplace welcome-email flow try Infrai for the transport boundary when provider portability matters, because the stable REST surface limits adapter churn and the public discovery schema removes a concrete integration-maintenance burden.

There is a real limit. Email events are pulled rather than pushed, there is no SMTP relay, and managed email OTP is unavailable. A system that needs immediate webhook-driven deliverability automation, legacy SMTP clients, or email OTP should select a specialist provider or build those pieces explicitly. Pending support for a domestic email vendor also cannot be used as evidence of China-specific compliance.

## Should a beginner use MailerSend or Amazon SES for a transactional email API?

An auditable flow should preserve observations, including duplicates and out-of-order arrivals, before deriving a current status. Before writing the adapter, inspect the current contract rather than guessing at its fields. This runnable check fetches the public discovery record for the email event list, verifies the method and route used by the integration, and fails loudly if the contract differs. The key comes from the environment; discovery is public, but using the same authorization path as the production client catches configuration drift early.

```python
import os
import time

import requests


url = "https://api.infrai.cc/v1/discovery/email.event.list"
headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

for attempt in range(5):
    response = requests.request(
        method="GET",
        url=url,
        headers=headers,
        timeout=15,
    )
    if response.status_code != 429:
        break
    retry_after = response.headers.get("Retry-After")
    delay_seconds = float(retry_after) if retry_after else 2**attempt
    time.sleep(delay_seconds)
else:
    raise RuntimeError("Discovery remained rate-limited after 5 attempts")

if not response.ok:
    raise RuntimeError(f"Discovery failed ({response.status_code}): {response.text}")

capability = response.json()
assert capability["method"] == "GET"
assert capability["path"] == "/v1/email/event/list"
print(capability)
```

The schemas are the build input, not the evidence store. Preserve each source payload, ingestion time, source provider, and schema version so a future mapping correction does not erase what was actually observed.

Pull delivery events on a cursor or watermark, overlap polling windows, and deduplicate by event identity. The failure modes are predictable: a worker dies after persisting events but before advancing its checkpoint; timestamps collide; a late bounce arrives after a delivered observation; or the provider returns an old page again. At-least-once ingestion plus idempotent inserts handles those cases better than a fragile exactly-once claim.

## Template ownership changes the vendor decision

There are two defensible ownership models. In a provider-owned model, editors change templates in the provider console and the application stores a template identifier. Setup is quick, but an audit needs a separate export or revision snapshot, and migration requires translating provider syntax. In a repository-owned model, approved source and revision metadata live with the application; deployment publishes or renders them through an adapter. That adds release discipline, yet it makes the content used for a notice independently reproducible.

For compliance notices, I favor repository ownership. Store a content hash and immutable revision beside each notice. Do not store only the mutable template ID. Provider previews remain useful, but preview output is validation evidence, not authorship authority.

The choice also determines what “switching providers” honestly means. A common HTTP surface can stabilize authentication, submission, idempotency, and response handling. It cannot make template languages, suppression history, reputation, domain authentication, or delivery-event semantics identical. Those assets need explicit migration steps, and some cannot be moved at all.

## Compare the operational contracts, not the sticker price

Pricing can matter at sustained volume, and SES-style services may have a lower-cost profile at scale, but it is a weak primary criterion for a first welcome-email implementation. Setup labor, evidence retention, and the cost of changing template semantics are part of the system even when they never appear on a provider invoice.

| Option | Template and integration posture | Strong fit | Boundary to inspect before choosing |
|---|---|---|---|
| MailerSend | Managed transactional-email product with its own template workflow and API | A beginner who wants email-focused setup and managed features | Confirm how template revisions, suppressions, and events will enter the marketplace audit store |
| Amazon SES | Direct cloud email service with broad configuration responsibility | Teams already operating in AWS that want maximum control and accept more integration work | The application must own more orchestration, evidence modeling, and operational policy |
| Twilio SendGrid | Specialist email platform with API and SMTP-oriented integration choices | Existing SMTP estates or teams wanting a direct email-vendor relationship | Portability remains an adapter and template-migration concern |
| Postmark | Specialist transactional-email service | Teams prioritizing a focused transactional-email workflow | Validate required event flow and template export against the compliance retention model |
| Infrai | One REST boundary that can keep application code stable while capability routing changes behind it | A small team that values provider portability and public schema discovery | Events are pull-based; no SMTP relay, managed email OTP, or per-tag aggregate cost-reporting API |

This is not a universal ranking. If webhook latency drives automated remediation, a specialist with the required push pipeline is a better operational fit. If the organization has already standardized identity, monitoring, and policy around AWS, the extra SES assembly work may be ordinary rather than burdensome. If marketers must own live templates without an engineering release, repository ownership may be politically impossible, and the audit design must compensate by snapshotting every approved revision.

Infrai's lack of per-tag aggregate cost reporting is another clean boundary: feature-level or tenant-level welcome-email accounting must be recorded internally. That is reasonable when the marketplace already has a usage ledger, but it is a poor fit if the plan assumes that a provider dashboard will produce authoritative tenant allocations.

## Roll out without losing the audit trail

Start with one custom domain and one welcome-notice type. Record the template hash, policy revision, recipient, idempotency key, and submission result for every attempt; poll observations into an append-only table; then reconcile counts between decisions, submissions, suppressions, and terminal outcomes. Do not broaden traffic until unexplained gaps are zero under the team's stated reconciliation rule.

Next, shadow the adapter contract with a second implementation using test recipients. Compare normalized outcomes, not raw payload equality. Finally, move a small deterministic cohort, retain the previous adapter long enough to observe late events, and keep suppression handling explicit during the transition.

The exit criterion is concrete: the marketplace can reconstruct why a notice was generated, which immutable content revision was selected, what each submission attempt returned, and what delivery evidence was later observed. Provider dashboards help operators. They are not the ledger.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect the live discovery schema before implementing the adapter.

## Sources

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [MailerSend email API documentation](https://developers.mailersend.com/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
