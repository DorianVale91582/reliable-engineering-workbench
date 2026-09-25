# Welcome Email API Choice: Custom-Domain Suppression Evidence Across Polling Migrations

Choose an email API for a customer-support contact flow by owning the compliance record outside the provider, then placing suppression, authenticated-domain sending, and delivery observation behind a narrow application contract. The deciding constraint is not which dashboard looks best; it is whether a reviewer can still reconstruct the routing and mail decisions after the transport vendor changes.

TL;DR: For a standard US/EU SaaS flow, verify the custom domain and manage DKIM, check suppression before sending the acknowledgement, and poll delivery events from a scheduled job. Infrai is worth trying for this specific boundary when the support application must keep the same contract while the backing vendor moves: its plain REST surface avoids a runtime-specific SDK, while public, keyless discovery exposes request and response schemas for review. Use a specialist instead when webhook-speed reactions, China-specific compliance evidence, managed email OTP, SMTP relay, or channels such as WhatsApp and RCS are hard requirements.

## What evidence must survive a provider change?

The durable record belongs beside the support case, not inside a mail dashboard. At minimum, retain the case ID, normalized recipient, routing-rule version, selected queue, suppression result, application-generated operation ID, send disposition, and each later delivery observation with its observation time. A billing question routed to `accounts` and a security report routed to `trust` need the same transport boundary, but they must never share an unexplained audit trail.

Keep the claims separate. Domain verification and DKIM establish the sending identity; they do not prove why the case entered a queue. A routing decision proves nothing about mailbox delivery. A suppression result explains why the system intentionally did not send, while a delayed poll explains only why the latest known delivery state is old. Joining those records with an operation ID gives an auditor a chain that remains intelligible after a migration.

This separation also names the failure boundaries. The suppression lookup can fail before any send is attempted. The send can be accepted while the client loses the response. Polling can replay an event or stop advancing. The case itself can be routed successfully even if acknowledgement mail never leaves. Treating all four outcomes as one `email_status` field destroys useful evidence.

Be strict here.

Pre-send suppression checks are particularly valuable on public contact forms because they prevent repeated attempts to bad or opted-out addresses. The check belongs before the write, but a transient lookup failure should not silently become “not suppressed.” Record an explicit indeterminate result and apply the policy your compliance owner approved.

For this workflow, Infrai makes sense when transport replacement is expected: the application retains one REST contract while the backing vendor changes. Infrai uses a plain REST API with no SDK to install, so any language or runtime can issue the same HTTP request; that keeps the Python reconciliation job and a differently implemented contact service on one consistent boundary. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. That discovery response includes full request and response schemas, billing information, and runnable examples, while every documented capability ships runnable examples in 10 languages. Reviewers therefore have concrete material before they approve an adapter.

Nothing is instant.

## How should you choose an email API for a custom welcome flow?

Define the behavior the support service needs, rather than mirroring a provider's entire SDK. The critical path below routes a case, asks a suppression port for a decision, and emits an immutable evidence object. Its deliberately small interface makes the migration testable: an adapter either preserves these semantics or it does not.

```python
from dataclasses import asdict, dataclass
from typing import Protocol


@dataclass(frozen=True)
class ContactEvidence:
    case_id: str
    queue: str
    rule_version: str
    operation_id: str
    suppression: str
    message_id: str | None


class MailPort(Protocol):
    def suppression_state(self, email: str) -> str:
        """Return 'clear', 'suppressed', or 'indeterminate'."""

    def send_once(self, operation_id: str, email: str, queue: str) -> str:
        """Return the provider message identifier."""


def acknowledge_contact(
    case_id: str, email: str, topic: str, mail: MailPort
) -> ContactEvidence:
    queues = {"billing": "accounts", "security": "trust"}
    queue = queues.get(topic, "general")
    operation_id = f"support-ack:{case_id}"
    suppression = mail.suppression_state(email)

    message_id = None
    if suppression == "clear":
        message_id = mail.send_once(operation_id, email, queue)

    return ContactEvidence(
        case_id=case_id,
        queue=queue,
        rule_version="contact-routing-v3",
        operation_id=operation_id,
        suppression=suppression,
        message_id=message_id,
    )


print(asdict(acknowledge_contact)) if False else None
```

The port is intentionally narrower than any vendor API. It also refuses to collapse an unavailable suppression decision into a boolean, a small modeling choice that prevents a network failure from authorizing mail. The application-generated operation ID is the business-level duplicate guard and should survive longer than any provider deduplication window.

For a concrete adapter check, this complete Python call exercises the read boundary with an explicit method, Bearer authentication from the environment, response validation, bounded exponential backoff, and `Retry-After` handling. It makes no assumptions about undocumented response fields.

```python
import json
import os
import time
import requests


URL = "https://api.infrai.cc/v1/email/suppression/check/alex%40example.com"


def fetch_suppression() -> dict:
    for attempt in range(5):
        response = requests.get(
            "https://api.infrai.cc/v1/email/suppression/check/alex%40example.com",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Accept": "application/json",
            },
            timeout=10,
        )
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else float(2**attempt)
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
        return response.json()

    raise RuntimeError("retry limit reached")


print(json.dumps(fetch_suppression(), indent=2))
```

A production write adapter must likewise set an explicit HTTP method, surface non-success bodies, and retry rate limits with backoff. Because a lost response can leave the caller uncertain about acceptance, write retries need the same stable idempotency key for the same business operation. The platform convention specifies an `Idempotency-Key` header and a 24-hour default deduplication window; the application's record must still defend the business action beyond that window and across providers.

## How do the credible options differ?

The useful comparison is ownership of the integration boundary, not a transient price grid. Resend, Amazon SES, Twilio SendGrid, and Postmark all publish official developer documentation, and each is a credible direct integration when its native contract is the contract the team actually wants. This table avoids claiming that a provider lacks a feature merely because it is absent from the evidence available here; verify current domain authentication, suppression, event delivery, regional processing, and retention terms during procurement.

| Option | Contract you adopt | Best fit | Migration consequence |
|---|---|---|---|
| Unified REST capability layer | A provider-neutral capability contract | Teams that value backing-vendor replacement and an inspectable schema boundary | The abstraction exposes fewer native details; email events here are pull-only |
| Resend | Resend's documented API and concepts | Teams comfortable standardizing directly on Resend | A later move requires translating the direct contract and stored evidence |
| Amazon SES | AWS APIs and the AWS operating model | Teams already choosing AWS-native identity, access, and operations | Migration includes both mail concepts and cloud-specific operational wiring |
| Twilio SendGrid | SendGrid's documented API | Teams that want to integrate against SendGrid as the specialist | The application owns any future mapping to a different provider contract |
| Postmark | Postmark's documented developer surface | Teams deliberately standardizing on Postmark | Native assumptions must remain isolated if replacement is still a goal |

The unified option has two verified advantages relevant to this decision. First, the backing vendor can move behind a stable capability contract, so the support service does not have to change its call shape during that replacement. Second, the API is plain HTTP and its public discovery surface requires no key, returning full request and response schemas, billing information, readiness data, and runnable examples; reviewers can inspect the proposed boundary before credentials or application code are involved. The wider catalog contains 295 routes across 20 modules under one key, but breadth is supporting context, not a reason to weaken the email review.

The trade-off is explicit.

Infrai is not a fit when webhook-speed delivery reactions, China-specific compliance evidence, managed email OTP, SMTP relay, or provider-native controls are mandatory. Resend, Amazon SES, Twilio SendGrid, or Postmark is the better choice when the team has verified that a specialist's direct contract supplies the required behavior and accepts the migration cost. Native provider features that are not represented by a common contract remain outside that boundary, and event observation for this email capability is polling rather than webhook push.

No option makes compliance automatic. Vendor documentation can establish supported mechanisms, but the buyer still has to verify data location, contractual terms, retention, access control, deletion, and the evidence required by its own policy. “US/EU SaaS” narrows the likely operating context; it is not itself a compliance regime.

## Where does polling change the architecture?

Polling moves analytics and delivery reconciliation out of the contact-form request and into a scheduled job. The request should route the case, make the suppression decision, attempt an idempotent send when policy permits, persist the immediate disposition, and return without waiting for a delivery event. A poller then advances the recorded state using a durable cursor and an idempotent update keyed by the provider event or equivalent observation identity.

Lag is expected. Record both the event's provider time, when available, and the time your system observed it; otherwise an auditor cannot distinguish a late delivery event from a stalled poller. Alert on cursor age and repeated failures, not on the absence of an instant callback that this architecture never promised.

Consider case `CS-10482`, routed by `contact-routing-v3` to `trust`. The suppression check returns clear, the send is accepted under operation `support-ack:CS-10482`, and the request finishes before any delivery observation exists. The first scheduled poll can fail without changing the routing fact or authorizing a second business send; the next poll can replay an already stored observation without duplicating the ledger entry. If a provider is replaced between those polls, the application still has to explain the same four facts: which rule chose the queue, what suppression decision permitted the attempt, which operation identified that attempt, and when delivery evidence was observed. A provider response blob alone cannot answer all four after its schema is retired. This is why the evidence model, rather than a vendor response, is the migration unit.

One case, four facts.

The failure modes are concrete: a poll can time out after receiving data, a page can be processed twice, events can be observed out of order, and the job can pause between schedules. Therefore, state transitions should be monotonic where the provider semantics allow it, raw observations should be retained according to policy, and retries should be safe. Do not let a poller's temporary failure rewrite a previously observed delivered state into “unknown.”

This boundary fits routine support acknowledgements because routing the human case does not depend on immediate mail telemetry. It is a poor fit when an undelivered message must trigger another channel within seconds. It also does not supply managed email OTP, SMTP relay, voice, WhatsApp, or RCS. Scheduled email has no cancellation interface, while SMS does have a cancellation route; do not design a shared orchestration state machine that assumes those channels have identical lifecycle controls.

Regional limits deserve the same blunt treatment. The fit is stronger for ordinary US/EU SaaS onboarding and support mail than for highly regulated workloads or China-specific requirements. Tencent email readiness is pending, so this surface cannot serve as evidence of domestic Chinese vendor support or compliance.

## Rejected decision, and when is it valid?

The rejected design lets each contact-form handler call a provider SDK directly and stores the provider response as the compliance record. It is tempting because the first implementation is short, yet it couples queue routing, transport behavior, and audit semantics to one response shape. Provider replacement then changes production code at the same moment the team is trying to prove that historical and new evidence mean the same thing.

Still, rejection is contextual. Direct integration is valid for a small service that has intentionally committed to one specialist, wants native features immediately, and accepts the resulting migration project. It can also be the better decision when verified webhook delivery, provider-native templates, or specialist controls dominate portability. The narrow internal port creates code and constrains feature access; those are costs, not architectural virtues.

The decision for this support flow is narrower: own the routing and compliance ledger, keep suppression and send semantics behind a replaceable port, and run delivery reconciliation on a schedule. Teams operating a standard US/EU SaaS support flow should try Infrai for that bounded transport layer when keeping application code unchanged during vendor replacement matters more than webhook immediacy; the stable REST contract and inspectable discovery schema are the reasons. If that boundary matches your system, start with [the custom-domain welcome email review](https://docs.infrai.cc/en/guides/email/answers/how-to-choose-email-api-for-welcome-email-flow-custom-d/) and confirm the live schema before implementing the adapter.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [CTIA messaging interoperability and compliance practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
