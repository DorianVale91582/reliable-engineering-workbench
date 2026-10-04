# Python SMS OTP Threat Model: NIST, GDPR, PSD2, SIM-Swap, Phishing Risks

Decision rule: SMS OTP is not enough for GDPR compliance, PSD2 strong customer authentication, or a reliable pharmacy refill login by itself. Treat it as a restricted authentication method, never as proof that a message, a financial report, or its attachment was delivered to the intended person; use it only after a documented risk assessment, and put higher-risk actions behind a phishing-resistant authenticator.

**Short answer:** SMS OTP can participate in a compliant design, but it is not enough by itself to establish GDPR, PSD2, NIST, or US health-privacy compliance. Those regimes regulate different things. GDPR asks for risk-appropriate security and data minimization; PSD2 strong customer authentication normally requires two independent factors and, for remote electronic payments, dynamic linking; NIST treats the public switched telephone network as a restricted authenticator and says phishing resistance must be available at AAL2. Pharmacy refill messages may also carry protected health information under US health-privacy rules. An OTP does not satisfy those surrounding duties, and it does nothing for email attachment integrity or delivery evidence.

This architecture decision record covers one deliberately awkward boundary: a fintech service generates a report for a health-spending account, emails the report as an attachment, and may send a pharmacy refill alert by SMS. Delivery reliability is the primary decision axis. Authentication, authorization, document delivery, and notification remain separate because combining them creates evidence that looks convenient but proves very little.

## Can SMS OTP meet GDPR, PSD2, and NIST compliance?

Four invariants define the design. First, no refill detail, medicine name, account balance, or report content appears in an SMS; the alert is a minimal prompt to visit an authenticated channel. GDPR Article 5 requires data minimization, while the HIPAA Privacy Rule permits uses and disclosures for treatment and other defined purposes rather than making every health-related text automatically acceptable. The exact US obligation depends on whether the sender is a covered entity or business associate and on the message content, so counsel and the privacy officer must classify the flow.

Second, OTP verification proves possession of a routed number at one moment. It does not prove the subscriber's civil identity, defeat real-time phishing, or show that the same person still controls the number after a SIM change. NIST SP 800-63B calls use of the PSTN for out-of-band authentication restricted and directs verifiers to consider risk indicators such as SIM change and number porting. A design that labels SMS as universally secure 2FA has already erased the failure mode it most needs to monitor.

Third, the email attachment is immutable after generation: store its content digest, report identifier, recipient authorization decision, and generation time before attempting delivery. Do not treat an SMTP acceptance response, a transport webhook, or an opened tracking pixel as proof that the authorized human received and read the report. RFC 5321 defines SMTP transfer behavior; it does not turn mailbox acceptance into end-user identity evidence.

Fourth, retries are idempotent. A queue retry may send the same logical notification again, but it must not create a second report, rotate the underlying authorization, or record a duplicate event as a second successful business outcome.

This sounds mundane. It is where reliability claims usually become accounting fiction.

The failure boundaries matter too: the authentication service decides whether a session satisfies policy; the report service renders bytes and records a digest; the delivery worker transports an already-authorized artifact; and an audit stream records state transitions. No delivery callback grants access. No OTP result marks an email delivered.

Transport is not identity.

## Compare the options by evidence, not convenience

| Pattern | Authentication properties | Delivery evidence | Main failure modes | Appropriate use |
|---|---|---|---|---|
| SMS OTP as the only login factor | Possession of a routed telephone number; restricted under NIST guidance | None for email or SMS content | SIM swap, port-out, recycled numbers, real-time phishing, delayed messages | Low-risk notification enrollment after a documented assessment |
| Password plus SMS OTP | Two entries, but the OTP is still phishable and depends on telephone routing | None for the attached report | Credential phishing can capture both factors; recovery can collapse them into one channel | Transitional access where policy accepts a restricted factor and safer alternatives are offered |
| Phishing-resistant authenticator plus authenticated retrieval | Cryptographic authentication can bind the ceremony to the verifier | Application records retrieval of a specific immutable object | Device loss and recovery abuse remain; notifications can still be delayed | Higher-risk account access and report retrieval |
| Email attachment plus generic status alert | Authentication happens before report generation | SMTP and downstream events describe transport, not human receipt | Forwarding, wrong address, mailbox compromise, filtering, scanning delays | Reports whose classification permits attachments, with controlled retrieval as fallback |

PSD2 needs its own row in the decision process, not a compliance sticker pasted onto SMS. Article 97 of the directive requires strong customer authentication for account access, electronic payment initiation, and certain remote actions. The associated regulatory technical standards define two or more elements categorized as knowledge, possession, and inherence, require their independence, and require dynamic linking for remote electronic payment transactions. A texted code might contribute a possession element in a particular implementation, but a pharmacy reminder is not thereby a PSD2 transaction, and a code that is not bound to a payment amount and payee does not create dynamic linking. Scope the action first.

GDPR is different again. Articles 5, 25, and 32 make purpose limitation, data minimization, protection by design, and risk-appropriate security relevant to the whole processing system. They do not certify a channel called SMS. Records of consent or another lawful basis, retention limits, processor arrangements, access controls, incident handling, and protection of message content remain outside the OTP exchange.

## Critical path: record intent before transport

The critical path below is intentionally generic Python. It assumes authentication and authorization have already happened, persists a stable message intent before attempting transport, and records transport outcomes without pretending they are user receipt. Concrete storage and queue implementations must provide atomic uniqueness for `intent_id`; an in-memory example would hide the consistency requirement.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Protocol


@dataclass(frozen=True)
class DeliveryIntent:
    intent_id: str
    account_id: str
    report_id: str
    recipient: str
    attachment_digest: str


class IntentStore(Protocol):
    def insert_once(self, intent: DeliveryIntent) -> bool: ...
    def mark_attempt(self, intent_id: str, transport_id: str) -> None: ...
    def mark_accepted(self, intent_id: str) -> None: ...
    def mark_retryable(self, intent_id: str, reason: str) -> None: ...


class MailTransport(Protocol):
    def send_attachment(
        self, *, recipient: str, filename: str, content: bytes, idempotency_key: str
    ) -> str: ...


def deliver_report(
    intent: DeliveryIntent,
    report_bytes: bytes,
    store: IntentStore,
    mail: MailTransport,
) -> None:
    actual_digest = sha256(report_bytes).hexdigest()
    if actual_digest != intent.attachment_digest:
        raise ValueError("report digest mismatch")

    if not store.insert_once(intent):
        return

    try:
        transport_id = mail.send_attachment(
            recipient=intent.recipient,
            filename=f"report-{intent.report_id}.pdf",
            content=report_bytes,
            idempotency_key=intent.intent_id,
        )
        store.mark_attempt(intent.intent_id, transport_id)
        store.mark_accepted(intent.intent_id)
    except TimeoutError:
        store.mark_retryable(intent.intent_id, "transport_timeout")
        raise
```

There is a sharp edge in that example: a timeout can occur after the transport accepted the message but before the worker received the response. The retry therefore needs the same idempotency key, and reconciliation must consume transport events using the recorded intent. If the selected transport offers no idempotent submission or stable message identifier, duplicates are an expected failure mode; the user-facing copy and audit model should admit that possibility.

Consider the full timeout sequence. The worker writes one intent, submits one attachment, and loses the response after the remote mail system has accepted the bytes; thirty seconds later, a queue lease expires and another worker sees the same job. Without atomic uniqueness and a transport idempotency key, the system can send twice, while a dashboard may still count the first attempt as a failure and the second as a success. With those controls, the second worker can reconcile the stable transport identifier instead. The trade-off is extra state and a reconciliation path in exchange for bounded duplication risk and evidence that survives a process crash. This is also why a generic retry count, even a precise number such as three attempts, is not a reliability guarantee: the location of the failure matters more than the count.

Order matters.

Use a state machine with narrow meanings: `created`, `submitted`, `accepted`, `temporarily_failed`, `permanently_failed`, and, where independently observable, `retrieved`. Do not call the SMTP state `delivered_to_person`. Put the report digest and policy version in the audit record, but keep OTP values, report contents, and unnecessary phone data out of logs. Retention for operational evidence should be set by an explicit legal and business schedule rather than by whatever the log platform happens to retain.

Operational alerts should distinguish queue age, temporary failures, permanent failures, duplicate submissions, and callback gaps. A single delivery-rate graph conceals the incidents that matter. Test the uncomfortable transitions: accepted-then-timeout, callback-before-worker-commit, recycled telephone number, attachment rejected for size or malware scanning, expired report authorization, and a user who changes both email and phone during recovery.

## Why reject SMS-only login here?

The rejected option is SMS OTP as the sole gate for opening a refill detail or a financial report. Its attraction is obvious: no authenticator enrollment ceremony, broad handset reach, and one familiar recovery route. Those are usability properties, not a complete security argument. SIM swaps and number reassignment attack possession, phishing can relay a fresh code, and carrier filtering makes authentication availability depend on a transport designed for messaging rather than high-assurance identity.

It still has a valid use case. A content-minimized alert such as "A new notice is available" can wake up a user who then enters an authenticated application, provided the organization has established the message's lawful basis, honored messaging consent and opt-out obligations where applicable, and avoided sensitive content in lock-screen previews. CTIA messaging principles are useful operational guidance for consumer consent and messaging practices, but they are industry guidance rather than a substitute for GDPR, PSD2, HIPAA, FCC, or state-law analysis.

For authentication, offer a phishing-resistant method where NIST AAL2 applies, maintain recovery that does not quietly fall back to the weakest channel, and evaluate PSTN risk signals when SMS remains available. For delivery, expose the report through authenticated retrieval even if policy also permits an attachment; keep the attachment path observable and replayable from a stable intent. The decisive distinction is small but durable: an alert says that something happened, an authenticator supports a session decision, and a delivery record describes transport. None should impersonate the others.

## References

- NIST, *Digital Identity Guidelines: Authentication and Authenticator Management (SP 800-63B)*: https://pages.nist.gov/800-63-3/sp800-63b.html
- Regulation (EU) 2016/679 (GDPR), Articles 5, 25, and 32: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- Directive (EU) 2015/2366 (PSD2), Article 97: https://eur-lex.europa.eu/eli/dir/2015/2366/oj
- Commission Delegated Regulation (EU) 2018/389 on strong customer authentication and dynamic linking: https://eur-lex.europa.eu/eli/reg_del/2018/389/oj
- US Department of Health and Human Services, HIPAA Privacy Rule summary: https://www.hhs.gov/hipaa/for-professionals/privacy/laws-regulations/index.html
- RFC 5321, *Simple Mail Transfer Protocol*: https://datatracker.ietf.org/doc/html/rfc5321
- RFC 7208, *Sender Policy Framework (SPF)*: https://datatracker.ietf.org/doc/html/rfc7208
- CTIA, messaging interoperability principles and best practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
