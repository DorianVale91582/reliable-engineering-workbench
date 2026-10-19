# Freight SMS OTP Login API: Cooldown Auditing and Rate Limits

Short answer: for a logistics portal that gates an auditable compliance notice, keep the integration small: one service owns OTP issuance and verification, one durable record links the successful authentication decision to the notice version, and transient resend and failure events expire on a stated schedule. The dominant storage term is usually not the six-digit code. It is the repeated event trail around requests, delivery updates, retries, and rejected guesses. Model that term before choosing an API.

A useful first estimate is `R x E x B x D`: authentication requests per day, events retained per request, stored bytes per event after indexes and replicas, and retention days. If a request generates one issue event, two delivery updates, one resend, and three failed checks, `E` is 7 rather than 1. This is an illustrative workload, not a benchmark. Cutting `D` for low-value attempt telemetry moves the large term; shaving a few bytes from the OTP representation does not.

The durable result should say which principal passed which policy, when the decision occurred, and which immutable notice revision was presented. It should not preserve a reusable secret. Short-lived challenge state can disappear after expiry, while the notice acknowledgement and the minimum evidence needed for review follow the organization's retention policy.

Retries multiply records.

## What should survive an audit?

Treat OTP delivery, authentication, notice presentation, and notice acknowledgement as different facts. A carrier accepting a message is not proof that a person read it. A correct code establishes control of a channel under the system's policy; it does not establish agreement with an unseen document. Collapsing those states into a single `delivered=true` field creates an attractive record that cannot answer a careful auditor's next question.

Use append-only identifiers to join the facts without copying message bodies through every table. A challenge row needs a random identifier, a verifier derived from the code, policy timestamps, an attempt count, and a terminal state. The compliance record needs the authenticated subject, the notice digest or immutable version identifier, the acknowledgement time, and the authentication decision identifier. Delivery-provider payloads belong behind a normalization boundary because their status vocabularies and callback shapes are integration details, not the audit model.

**Store the decision, not the secret.** NIST SP 800-63B requires verifiers to rate-limit failed authentication attempts and treats use of the public switched telephone network for out-of-band authentication as restricted. That makes SMS a risk-managed channel, not durable evidence by itself. The record must preserve the policy outcome while the OTP value and unnecessary phone-number copies are removed.

## How much event history earns its retention cost?

Retention should follow evidentiary value. The table below separates records that answer a compliance question from operational exhaust that mainly helps with short-window diagnosis.

| Record class | Operational purpose | Retention decision | Failure mode if removed too early |
|---|---|---|---|
| Challenge state | Enforce expiry, cooldown, and attempt limits | Keep only through the challenge lifetime plus bounded reconciliation time | Concurrent workers can accept a stale or superseded challenge |
| Normalized delivery events | Diagnose handoff and callback ordering | Keep a short, explicit window or aggregate counts | A disputed carrier handoff becomes harder to investigate |
| Authentication decision | Explain why access was granted or denied | Retain under the access-evidence policy | The notice record loses its authentication link |
| Notice revision and acknowledgement | Show exactly what was presented and accepted | Retain under the applicable records policy | The system can show access but not the content acknowledged |
| Raw callback payload | Debug an adapter | Delete after normalization and reconciliation | Provider-specific details are unavailable during a late dispute |

There is no universal day count in the cited authentication guidance, so inventing one would be false precision. Legal and security owners must set the period for the jurisdiction and notice type. Engineering still has a concrete lever: give each record class its own expiry, document the deletion job, and test that a legal hold or equivalent policy state prevents deletion where required.

What do we deliberately stop keeping? Raw callbacks, expired challenge rows, full phone numbers in general-purpose logs, and per-attempt traces after their diagnostic window. The cost is real: a late complaint may be explainable only from normalized status and aggregate telemetry, not from the original carrier payload. Accept that loss explicitly, or lengthen the relevant window; accidental forever-retention is not a defensible third choice.

## How should an SMS OTP login API verify codes?

The following Python sketch shows the storage contract, although the same transaction boundary applies to a Node.js service. Values such as a 60-second cooldown and five verification attempts are example policy inputs, not standards. The critical property is atomicity: concurrent requests must not bypass either limit, and a successful decision must be linked to the notice revision in the same durable workflow.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Protocol


@dataclass(frozen=True)
class OtpPolicy:
    resend_cooldown: timedelta = timedelta(seconds=60)
    challenge_ttl: timedelta = timedelta(minutes=5)
    max_attempts: int = 5


class ChallengeStore(Protocol):
    def issue_if_allowed(
        self, subject_id: str, verifier: bytes, now: datetime, policy: OtpPolicy
    ) -> str: ...

    def consume_if_valid(
        self, challenge_id: str, candidate: str, now: datetime, policy: OtpPolicy
    ) -> str: ...


class EvidenceStore(Protocol):
    def acknowledge(
        self,
        subject_id: str,
        auth_decision_id: str,
        notice_revision: str,
        acknowledged_at: datetime,
    ) -> None: ...


def record_notice_access(
    challenges: ChallengeStore,
    evidence: EvidenceStore,
    challenge_id: str,
    candidate: str,
    subject_id: str,
    notice_revision: str,
    policy: OtpPolicy,
) -> None:
    now = datetime.now(timezone.utc)
    decision_id = challenges.consume_if_valid(challenge_id, candidate, now, policy)
    evidence.acknowledge(subject_id, decision_id, notice_revision, now)
```

The interface hides an important implementation requirement: `issue_if_allowed` must serialize decisions for the same subject or destination, and `consume_if_valid` must increment failures and transition success exactly once. A read followed by an unrelated write is insufficient under concurrency. Use the datastore's conditional update or transaction mechanism and make callback ingestion idempotent with a unique event identifier.

Consider the last permitted guess arriving at two application workers. Both workers read an attempt count of four; both compare a candidate; both then write five. Without a conditional update, two guesses occupied one policy slot, and an unlucky interleaving can also let success race with expiry or replacement. The same shape appears on resend: two requests both observe that the cooldown has elapsed, both create challenges, and whichever message arrives last may not correspond to the verifier the database considers current. A per-subject transaction, version check, or equivalent atomic datastore primitive must decide the winner. This is where I would spend integration effort first, because a polished message adapter cannot repair ambiguous authentication state.

One winner. One decision.

Do not log `candidate`. Do not return distinct errors for “unknown account” and “wrong code” if that distinction exposes account existence. Keep user-facing responses stable while recording a coarse internal reason code, and put per-subject controls beside broader destination, network, and system-capacity controls. A cooldown alone blocks a tight resend loop but does nothing about distributed attempts across many destinations.

## Integration effort is mostly failure handling

An adapter that sends a message is the easy part. The integration budget is consumed by duplicate callbacks, callbacks arriving out of order, timeouts with an unknown send outcome, destination normalization, consent and suppression rules, secret rotation, test environments, and reconciliation between a provider's status and the local challenge state. Those boundaries deserve contract tests before the first production notice.

Start with three failure injections: submit the same callback twice, deliver an older status after a newer one, and race two verification attempts against the last allowed attempt. Then test clock boundaries around cooldown and expiry with an injected clock. Deployment needs metrics for issuance outcomes, verification outcomes, throttling, callback lag, and normalization failures, but metric labels must not contain phone numbers or challenge identifiers because high-cardinality identifiers enlarge both the bill and the exposure surface.

DMARC applies to email-domain authentication and reporting, not SMS OTP verification. It becomes relevant only if the compliance workflow also sends notices or fallbacks by email. Keep that evidence path separate: an email authentication result can support investigation of domain handling, but it does not prove that a recipient read or acknowledged the notice.

The decision rule is blunt. Prefer the integration whose behavior can be mapped into the four local states you actually audit: issued, delivery-observed, authentication-decided, and notice-acknowledged. Count adapter code, reconciliation work, operational controls, and retained event volume. A short send call paired with ambiguous callbacks is still a large integration.

The trade-off is explicit.

## Ship the policy, then prove deletion

Before release, security should approve attempt and resend limits, records owners should approve retention by record class, and the application team should demonstrate atomic transitions under concurrency. Run restore tests for durable evidence and deletion tests for transient data. Both matter; a backup that silently resurrects expired OTP telemetry defeats the published retention model.

The final architecture is intentionally narrow: ephemeral challenge data, bounded operational events, and a durable link from an authentication decision to an immutable notice revision. This minimizes integration surface without pretending SMS delivery equals acknowledgement. It also makes the storage trade-off inspectable: keeping less raw history weakens late forensic reconstruction, while keeping every event indefinitely increases cost and data exposure without improving the meaning of the compliance record.

## Further reading

- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
