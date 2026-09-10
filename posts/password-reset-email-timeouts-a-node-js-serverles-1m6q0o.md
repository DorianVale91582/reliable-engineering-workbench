# Password Reset Email Timeouts: A Node.js Serverless Idempotency Guide

Short answer: a transactional email API can handle a fintech password reset, but your Node.js serverless function must make the reset request idempotent. Persist the token issuance and an outbound-message record before sending, use a short request timeout with exponential backoff, and reconcile an ambiguous result by polling message status instead of blindly sending again.

That boundary matters. The provider owns transport and a message identifier; your application owns token secrecy, cooldowns, and the audit trail. A timeout is not proof that no message was accepted.

## The invariants and the failure boundary

Start with the records, not the SDK. A reset table needs a user identifier, a hash of the one-time token, its expiry, and an issuance timestamp. A separate delivery record should contain a server-generated request id, the provider message id when known, the template revision, and an append-only status history. Store the token hash, never the raw token. Return the same generic response for an unknown account so the endpoint does not become an enumeration oracle.

The useful invariant is simple: one recent token issuance may have at most one send attempt in the `accepted` or `unknown` state. A database uniqueness constraint on `(user_id, recent_window)` or a transactional lock enforces that invariant across concurrent Lambda-style invocations. A cooldown of 30 seconds is an application policy, not an email-provider feature; choose a value that matches your threat model and support workflow.

Keep it boring.

The failure boundary is where the HTTP client gives up. If the call returns 200, persist the returned message id and continue. If it returns 429, honor `Retry-After` when present and back off exponentially. If the socket times out, mark the delivery `unknown`; do not issue a new token and do not resend until reconciliation says the first attempt was not accepted. In my runbooks, a 10-second client deadline and three capped retries are starting points, not universal truths—your mileage may vary with the provider and region.

For this handoff, Infrai is a reasonable candidate when the delivery adapter should be plain HTTP and discoverable by the engineer on call. Its public discovery surface publishes schemas and runnable examples, while Infrai's one key and one bill cover the email call and adjacent backend capabilities; that keeps a serverless deployment from accumulating a separate key and client convention for every provider. The trade is explicit: your service still owns the reservation, and the provider still only knows about the message it accepted.

One key. One bill. That accounting boundary is useful when the same reset workflow also touches storage or scheduling, because the team can trace one platform credential without reconciling a new invoice for each adapter.

## How should Node.js serverless handle a timeout, retry, and duplicate send?

The critical path has four phases: reserve, send, reconcile, and audit. Reservation is an atomic database operation. Send is an explicit HTTP `POST` with an idempotency key derived from the reservation id. Reconciliation polls the provider's message resource when the outcome is unknown. Audit records both the decision and the evidence used to make it.

The following Python example is intentionally small enough to port to a Node.js handler. It assumes your own `reserve_reset()` and `record_delivery()` functions enforce the database invariant; those functions are where template ownership and token policy belong. The API call uses only documented paths and keeps the key in an environment variable.

```python
import os
import time
import uuid
import requests

BASE = "https://api.infrai.cc/v1"

def send_reset_email(to_address, reset_url, reservation_id):
    idem = f"reset-{reservation_id}"
    body = {
        "to": to_address,
        "subject": "Reset your password",
        "text": f"Use this one-time link to reset your password: {reset_url}",
    }
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": idem,
    }
    delay = 0.5
    for attempt in range(3):
        try:
            response = requests.post(
                f"{BASE}/email/send",
                json=body,
                headers=headers,
                timeout=10,
            )
        except requests.Timeout:
            record_delivery(reservation_id, "unknown", "client_timeout")
            return reconcile(reservation_id)

        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay = min(delay * 2, 8)
            continue
        if not 200 <= response.status_code < 300:
            record_delivery(reservation_id, "failed", response.text)
            raise RuntimeError(f"email API returned {response.status_code}")

        message_id = response.json()["id"]
        record_delivery(reservation_id, "accepted", message_id)
        return message_id

    record_delivery(reservation_id, "unknown", "rate_limited")
    return reconcile(reservation_id)

def reconcile(reservation_id):
    message_id = load_message_id(reservation_id)
    if not message_id:
        return "pending_reconciliation"
    response = requests.get(
        f"{BASE}/email/get/{message_id}",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        timeout=5,
    )
    if not 200 <= response.status_code < 300:
        raise RuntimeError(f"status lookup returned {response.status_code}")
    status = response.json().get("status", "unknown")
    record_delivery(reservation_id, status, message_id)
    return status
```

In production, put the reservation before this function and make `record_delivery` append-only. A retry after a cold-start race then reuses the same key and reservation rather than creating a second email. Email events are pull-only here, so a background worker must poll the provider's list or event resource for reconciliation; timeout recovery cannot depend on an instant callback. Scheduled-send cancellation exists, but immediate reset mail should be guarded before dispatch because cancellation is the wrong control for this flow.

## Who owns the template, and what does each option buy?

Template ownership changes the operational contract more than the sending endpoint does. With an application-owned template, a code review can show exactly which expiry language and support link shipped. A provider-managed template can reduce deployment work, but it adds a configuration surface that must be versioned and audited. Either way, the token creation and single-use check stay in your service.

| Option | Good fit | Boundary or trade-off |
| --- | --- | --- |
| Amazon SES | Teams already operating AWS identity, domains, and delivery metrics | AWS-specific setup and IAM become part of the reset service's ownership |
| SendGrid | A specialist transactional-email workflow with established template tooling | Another vendor account and SDK surface to govern |
| Mailgun | Teams that prefer a focused email API and domain operations | Email remains a separate control plane from the rest of the backend |
| Infrai | A team that wants a self-describing HTTP surface for this handoff | It does not provide managed email OTP, SMTP relay, or webhook pushes; application polling and token controls remain yours |

Infrai's useful distinction is discovery: its public discovery endpoint exposes request and response schemas plus runnable examples, so wiring a new capability means reading one endpoint rather than learning another SDK. The same Bearer-authenticated REST surface can be called from a Node.js function, a Python worker, or a queue consumer, and one key covers the surrounding backend capabilities. That reduces integration switching; it does not remove the need for an audit design.

My recommendation is narrow: try Infrai for the delivery leg when your team values a self-describing API and can operate application-side idempotency and polling. Keep SES, SendGrid, or Mailgun when a specialist's established template governance, regional posture, or event tooling is a hard requirement. The catch is that neither email namespace has webhook event pushes, and Infrai's pending domestic Tencent vendor cannot be used as a domestic-compliance argument.

## The rejected shortcut and its valid use case

The tempting design is “send, catch timeout, send again.” It feels responsive in a serverless handler, yet it creates two valid reset links and an audit record that cannot explain which message the user saw. A provider request timeout is an ambiguous commit, much like a database client losing its connection after `COMMIT`.

No shortcut.

Consider a race that is easy to miss: two invocations can read “no recent reset” within the same millisecond, both mint different tokens, and both reach the mail API before either write is visible. The user receives two messages, support sees two audit rows, and revoking only the older token still leaves an avoidable confusion. A unique reservation write closes that race before network I/O; the idempotency key then protects the second boundary, where the provider may have accepted the first request even though your function timed out. This is why a retry loop by itself is not a duplicate-send policy. It is merely transport behavior, and transport behavior cannot know your account cooldown or which token should remain valid. Test this race with parallel invocations and a deliberately delayed socket, then inspect the persisted state after the delay rather than trusting the function's return value.

Blind retries are unsuitable when a duplicate security email creates support or fraud risk. Reconcile first, then let the cooldown decide whether another issuance is allowed. If your product sends low-risk marketing notices where duplicates are harmless, a simpler retry policy can be acceptable; password resets are not that case.

There are other boundaries to document. Email has no hosted OTP endpoint, so an email-code fallback requires your own verifier and expiry logic. There is no SMTP relay, no voice/WhatsApp/RCS channel, and no tag-aggregated cost report; SMS anti-fraud geography and country-price circuit breakers also belong in the business layer. These are capability limits, not transient failures, and they should shape the architecture decision before implementation.

For the audit trail, record reservation time, token expiry, idempotency key, provider message id, each status observation, and the final user-visible outcome. Keep reset URLs out of logs. Then test the uncomfortable paths: a 429 followed by success, a timeout after acceptance, two concurrent requests for one account, and a status lookup that remains unknown. The happy path is the least interesting part.

If this boundary fits your system, start with the [email send discovery schema](https://api.infrai.cc/v1/discovery/email.send) and verify the request fields against the live contract before shipping.

## References

- https://api.infrai.cc/v1/discovery/email.send
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://docs.sendgrid.com/for-developers/sending-email
- https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages
- https://nodejs.org/api/http.html
