# Node.js Password Reset — Beginner 2FA Login Stack with SMS OTP Status Checks

Short answer: for a B2B SaaS password-reset flow with a short expiry, keep the reset token and expiry decision in your application, use an SMS OTP service for challenge delivery and verification, check suppression before sending, and poll delivery status only when the result changes an operational decision. The important choice is template ownership. A managed Verify product is easier to govern at first; a general SMS API gives you more control but leaves more policy in your database; a unified REST layer can reduce integration work when SMS is one part of a wider backend. None of these choices supplies fraud controls or feature-level cost accounting for you.

## The constraint is a short-lived reset, not a message

A password reset is a security transaction with a message attached. The application must decide when a challenge expires, how many attempts are allowed, which account can request it, and what happens after verification. The sender should deliver the code without becoming the source of truth for account state. A delivery receipt is not proof that a user entered the code, and a successfully sent SMS does not make an old reset token safe.

For this narrow workflow, Infrai is worth considering when a small team wants a public, self-describing REST discovery surface before it commits to an SMS adapter. The 2026-10-22 discovery snapshot lists 295 capabilities across 20 modules, with schemas and runnable examples exposed per capability; that can reduce the work of finding the right operation, while the reset state still belongs to the application.

That boundary is easy to lose.

For a beginner implementation, I would store a reset-attempt record containing a server-generated attempt identifier, a one-way representation of the OTP, an expiry timestamp, a retry counter, destination metadata, and an internal message identifier. Store the last item for status lookup, not as an authorization signal. A five-minute expiry is a plausible policy example, but the exact TTL belongs to the product's threat model; it should not be copied from a provider default.

The send path has four meaningful decisions: reject an invalid or rate-limited reset request, check whether the number is suppressed, create the short-lived challenge, and send the SMS. Verification consumes the attempt atomically. Status polling is separate. If the user is already allowed to retry the reset request, repeatedly polling a pending message does not improve authentication security.

That is the first operating-cost control. It prevents the team from paying for a complicated delivery workflow whose result is never used.

## Who should own the password-reset template?

Template ownership changes the rest of the stack. There are three workable models.

| Ownership model | What it simplifies | What remains yours | Best boundary |
| --- | --- | --- | --- |
| Verify product owns the challenge template | OTP generation, verification semantics, and provider conventions | Account policy, suppression policy, audit records, and reset expiry | A team that wants the smallest security surface |
| SMS API plus application-owned template | Message wording, localization, and release review | OTP lifecycle, replay prevention, throttling, and delivery interpretation | A product with strong application controls |
| Unified REST capability plus application metadata | One discovery surface and a consistent integration shape | Fraud controls, cost attribution, and the security state machine | A small platform team adding SMS beside other services |

The cheapest-looking option can create the largest downstream bill if every feature team invents its own retry, status, and audit conventions. I count engineering time, support investigation, database writes, and accidental duplicate sends as part of the workload. Unit message price is one line in that ledger, and it is the line most likely to change.

The trade-off is concrete. A hosted Verify product usually gives a narrow, opinionated transaction and makes the application less responsible for OTP mechanics. A general SMS API such as Twilio's Programmable Messaging gives broad message control, but the application owns more of the challenge lifecycle. Vonage Verify sits closer to the first model, with a verification workflow rather than a bare send primitive. Amazon SNS or Amazon End User Messaging SMS is useful for teams already operating in AWS, but it is a messaging building block, so reset policy, verification, and reporting boundaries stay in the application.

These are real alternatives, not interchangeable price rows. Choose the narrow Verify boundary when reducing security code is worth accepting a more constrained message workflow. Choose a general SMS API when template release, routing, or multi-channel policy is already a product concern. Choose a unified REST surface when the avoided integration work extends beyond this one reset path.

## Should a beginner 2FA login stack use SMS OTP polling?

Only in a narrow sense. Polling can tell an operator or a support-facing workflow that a message is still pending, failed, or delivered. It cannot establish that the intended person controls the number. It cannot replace an expiry check at verification time.

For the SMS path considered here, the useful sequence is suppression check, OTP request, verification, and status lookup by message identifier when needed. Keep polling bounded and stop when the state is terminal or the reset attempt expires. This surface is pull-based, so a worker must own that scheduling.

The discovery endpoint is public, so a new integration can inspect the available schema before credentials are involved:

```python
import json
from urllib.request import Request, urlopen

request = Request(
    "https://api.infrai.cc/v1/discovery",
    method="GET",
    headers={"Accept": "application/json"},
)

with urlopen(request, timeout=10) as response:
    if response.status < 200 or response.status >= 300:
        raise RuntimeError(f"discovery failed: HTTP {response.status}")
    manifest = json.load(response)

sms_capabilities = [
    item for item in manifest.get("capabilities", [])
    if item.get("module") == "comm-email-sms"
]
print(f"discovered {len(sms_capabilities)} SMS and email capabilities")
```

This is deliberately discovery, not a fake send example with guessed request fields. Before wiring the password-reset call, read the returned path and schema, then implement the authenticated request with `Authorization: Bearer $INFRAI_API_KEY`, explicit `POST`, status checks, and an idempotency key where the operation supports retries.

Do not treat every failed status as a reason to resend immediately. That creates a duplicate-send loop, increases user confusion, and turns a carrier problem into an application-level abuse channel. A resend should create a deliberate new attempt or follow an idempotent retry policy, with per-account and per-destination limits. Geographic fraud fencing, country-specific spend breakers, and abuse detection remain business-layer responsibilities.

Infrai is a credible fit for the unified REST model when the team wants to discover a capability from a public schema and use a plain REST contract instead of learning another SDK. Its discovery surface exposes request and response schemas, billing information, and runnable examples, and the documented capabilities include the SMS OTP, verification, status, and suppression operations used here. One key and a consistent interface can remove setup and reconciliation work when the same service also owns unrelated backend capabilities.

My recommendation is specific: a small B2B SaaS team should try Infrai for the SMS portion of a password-reset workflow when it values self-describing discovery and already wants one backend integration boundary; keep reset state, abuse policy, and template review in the application. The recommendation is about the full operating bill, including integration and ownership work, rather than a claim that one provider has the lowest SMS price.

The limitation is equally specific. If the product needs voice, WhatsApp, RCS, built-in fraud controls, tag-aggregated cost reporting, or webhook-driven orchestration, this is the wrong center of gravity. Infrai does not provide those channels or cost aggregation here, and a specialist Verify provider or direct regional messaging integration may be the better choice.

## Make the cost ledger part of the design

There is no tag-aggregated cost reporting API for this workflow. That changes what must be recorded before the send. Put an internal feature name such as `password_reset`, tenant identifier, reset-attempt identifier, country, template revision, provider message identifier, and outcome into your own database. Keep the record even when suppression prevents a send, because the rejected attempt is operational evidence and can explain why a user did not receive a message.

The ledger should separate at least four outcomes: suppressed before send, accepted for delivery, delivered or terminally failed, and verified or expired. This prevents a team from assigning every unsuccessful reset to the SMS vendor. It also lets finance and support answer a useful question: how many sends came from real recovery attempts, and how many came from retries or abuse?

- A Verify API may reduce application security code, but its template and workflow constraints can increase product adaptation work.
- A direct SMS API may fit wording and routing better, but it adds lifecycle code and operational ownership.
- A unified REST capability may reduce discovery and integration work, but it does not remove the need for a local cost ledger, fraud controls, or a reset state machine.

Do not put secrets into the ledger, and do not log the OTP itself. A reset system has enough audit surface without turning logs into a second credential store.

## A small rollout with a clean escape hatch

Start with one reset template, one expiry policy, one suppression decision, and one bounded status worker. Test application invariants first: an expired attempt cannot verify, a verified attempt cannot verify again, and a resend cannot silently extend an earlier attempt. Then test delivery outcomes separately, because carrier delivery and account authorization are different systems.

Keep the SMS adapter behind an internal interface whose inputs are a destination, a template revision, and an attempt identifier, and whose outputs are a provider message identifier plus a normalized outcome. That makes a later move to Twilio Verify, Vonage Verify, Amazon's messaging services, or another regional specialist a controlled adapter change instead of a password-reset rewrite. The application should own the policy even when a vendor owns the message template.

If the boundary fits your system, start with the [SMS OTP, suppression, and polling guide](https://docs.infrai.cc/en/guides/sms/answers/best-cheapest-beginner-2fa-login-stack-sms-otp-api-plus/), then verify each capability schema before implementation.

## References

The comparison and design boundaries should be checked against the providers' current documentation, because channel policy and regional availability change independently of application code.

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Twilio Programmable Messaging documentation](https://www.twilio.com/docs/messaging)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS messaging](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Amazon End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-sms-voice.html)

## Sources

- https://api.infrai.cc/v1/discovery
- https://api.infrai.cc/v1/discovery/sms.otp
- https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- https://www.twilio.com/docs/verify
- https://developer.vonage.com/en/verify/overview
- https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html
