# Property Text Classification: Using a 4-Stage Node.js Boundary for Portable Moderation

**Short answer:** For Node.js implementing text classification using a language-model API, treat the model as a replaceable decision service, not as the owner of property-moderation policy. Keep the taxonomy, validation, evidence capture, retry policy, and human-review threshold in your application; place provider-specific request shaping behind one adapter. That boundary is more valuable than a clever prompt because a report that is misclassified can affect a tenant, a listing, or an escalation queue.

The practical decision rule is this: accept a provider only when the same input contract, normalized output, failure policy, and audit record can be exercised against at least two implementations without changing the moderation workflow. The model can suggest a label. It should not silently redefine what the label means.

Not every report should be automated.

## What must remain true when the provider changes?

Start with invariants. A moderation report has a stable internal shape: an identifier, the property or listing context, the report text, and the policy version used for classification. The classifier returns a finite label, a confidence-like score that your policy defines, a short rationale for a reviewer, and a decision such as `review`, `allow`, or `escalate`. Unknown labels, malformed JSON, missing fields, and unsupported policy versions are failures, not creative outputs.

The boundary also needs failure limits. A timeout is not a negative classification. A second attempt must not create a second moderation action. A provider response may be valid JSON and still be semantically wrong, such as a label outside the current taxonomy or a rationale that cites evidence absent from the report. Validation has to cover both syntax and meaning.

Here is the small contract I would put in an architecture record before choosing a provider:

| Boundary | Application owns | Adapter owns |
| --- | --- | --- |
| Input | Taxonomy, policy version, report text, redaction | Provider request shape and authentication |
| Output | Allowed labels, threshold policy, audit event | Provider response parsing and usage metadata |
| Failure | Retry budget, idempotency, human fallback | Timeout translation and provider error mapping |
| Change | Evaluation set and rollout decision | Model-specific parameters and schema features |

This is deliberately boring. Boring boundaries survive migrations.

## How should a report move through the classifier?

## How can Node.js implement text classification using a portable boundary?

The critical path is a four-stage pipeline: normalize, classify, validate, then route. Normalization removes fields the model does not need and records the policy version. Classification calls an adapter. Validation checks the returned object against the application contract. Routing applies the threshold and sends uncertain or failed cases to human review.

The example below shows the shape of that path without tying the workflow to one API. In a Node.js service, the same interfaces can be implemented with `async` functions; the important part is that the queue worker calls `classify_report`, not a provider SDK directly.

```python
from dataclasses import dataclass
from typing import Literal, Protocol

Label = Literal["safety", "habitability", "fraud", "noise", "other"]
Decision = Literal["review", "allow", "escalate"]

@dataclass(frozen=True)
class Report:
    report_id: str
    property_id: str
    text: str
    policy_version: str

@dataclass(frozen=True)
class Classification:
    label: Label
    score: float
    rationale: str

class Classifier(Protocol):
    def classify(self, report: Report) -> Classification: ...

ALLOWED_LABELS = {"safety", "habitability", "fraud", "noise", "other"}


def route(report: Report, classifier: Classifier) -> Decision:
    result = classifier.classify(report)
    if result.label not in ALLOWED_LABELS:
        raise ValueError("unknown moderation label")
    if not 0.0 <= result.score <= 1.0:
        raise ValueError("score outside application range")
    if result.score < 0.75:
        return "review"
    if result.label in {"safety", "fraud"}:
        return "escalate"
    return "allow"
```

The `0.75` value is an application policy example, not a universal truth. It belongs in a versioned configuration and should be selected from labeled reports, with false negatives treated as a separate risk from queue volume. Do not call a score a probability unless it has been calibrated and tested as one. The limitation is that a threshold cannot repair a taxonomy that omits an important class of report.

## Which failures belong in the human queue?

Property reports are short, messy, and consequential. A tenant may write “the lock is broken again,” while a landlord may describe the same event as a maintenance dispute. A classifier can identify likely categories, but it cannot erase ambiguity in the source text.

I would route these cases to review: an unknown label, invalid structured output, a timeout after the retry budget, a score near the threshold, a report containing a new policy term, or a rationale that does not quote or accurately summarize the report. The queue record should include the original report reference, policy version, provider adapter name, model identifier, request correlation ID, validation result, and timestamps. Store enough to reproduce the decision without storing more personal data than the review process needs.

Retries need an idempotency key derived from the report identifier and policy version. Otherwise, a worker restart can produce duplicate audit events or make two provider calls look like two independent decisions. Backoff is useful for transient transport errors; it does not repair a schema violation or a policy disagreement.

Observability should separate provider health from classifier quality. Track timeout and error rates, validation failures, label distribution, review rate, and disagreement with human decisions. A rising `other` rate can mean the taxonomy is stale. A stable error rate does not prove useful classification.

## What does portability cost in practice?

Portability is not achieved by replacing one URL. The three major API families commonly used for this kind of work expose different request and response conventions: OpenAI documents a Responses API, Anthropic documents a Messages API, and Google documents a Gemini API. Their adapters can all implement the same internal `Classifier` protocol, but prompt placement, structured-output support, token accounting, safety controls, and error taxonomies still require provider-specific code.

That code is a real cost and the central trade-off. The payoff is a testable decision surface. Use a small conformance suite containing ordinary reports, ambiguous reports, adversarial instructions embedded in report text, truncated inputs, and every allowed label. Run it against each adapter and compare normalized results, not raw response text. A provider switch is acceptable only when the resulting routing behavior stays within explicitly chosen bounds.

| Option | Useful boundary | Main risk |
| --- | --- | --- |
| Direct SDK calls in workers | One provider, a short-lived experiment | Policy and transport become inseparable |
| One internal adapter | Multiple providers or a long-lived queue | Adapter maintenance and feature lag |
| Self-hosted model behind the adapter | Data residency or controlled deployment | Operational load, model updates, and quality variance |

The correct choice depends on the organization’s controls and evaluation data. Provider portability does not mean every implementation must expose identical capabilities; it means the workflow has a known minimum contract and knows when a capability gap requires human review. The limitation of the adapter approach is ongoing maintenance: a team that cannot maintain conformance tests may reasonably choose one direct integration for a low-risk queue.

## The rejected option: let the model own routing

A tempting design asks the model to return a label and an action, then writes that action directly to the moderation system. I reject it for this workflow. It combines taxonomy, risk appetite, and operational authority in a response whose format and semantics can drift with a prompt or model change. It also makes a provider comparison unfair: one API may produce a plausible action while another exposes a validation failure that is easier to detect.

That design has a valid use case for low-risk triage where the result is only a search hint and a person sees every item. It is a poor boundary for property reports involving safety, habitability, fraud, or account action. Keep authority in deterministic application code, and make uncertainty visible.

The implementation question is therefore narrower than “which API classifies text best?” It is “can this moderation workflow preserve its policy, evidence, and fallback behavior while the classifier changes?” If the answer is no, the system is coupled at the wrong layer.

## References

- https://python.langchain.com/docs/integrations/chat/openai/
- https://platform.openai.com/docs/api-reference/responses
- https://docs.anthropic.com/en/api/messages
- https://ai.google.dev/api/generate-content
- https://www.nist.gov/itl/ai-risk-management-framework
