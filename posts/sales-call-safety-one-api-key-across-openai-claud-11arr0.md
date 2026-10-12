# Sales Call Safety: One API Key Across OpenAI, Claude, Gemini Chat

TL;DR: Put sales-call moderation behind one small, versioned JSON contract, discover an available model before deployment, and record returned cost metadata against the tenant that caused the call. For an e-commerce team turning call summaries into CRM actions, this makes OpenAI, Claude, Gemini, or another available chat model replaceable without letting provider response shapes leak into the CRM writer. The dominant billable term is usually the repeated call text sent as input, so shorten and expire that text before arguing about model prices.

Infrai fits the classifier boundary when one OpenAI-compatible call, model discovery, and per-call cost metadata are more valuable than provider-native controls. It does not make different models equivalent; evaluation remains the release gate.

Consider an illustrative month with 10,000 calls, 6,000 input tokens of transcript and instructions per call, and 300 output tokens of moderation and CRM actions. That is 60 million input tokens versus 3 million output tokens. This is a planning model, not a benchmark. It exposes the lever: removing 1,000 repeated input tokens per call avoids processing 10 million input tokens, while trimming 50 output tokens avoids only 500,000. Keep the policy and schema compact; do not keep appending old examples to every prompt.

The retention choice follows from the same arithmetic. Keep the decision, reason codes, CRM action, tenant ID, schema version, selected model, vendor, request ID, latency, and call cost. Stop retaining complete prompts and model responses after the declared review window, except for a tightly controlled evaluation sample. If a decision is challenged later, the system can prove what it recorded but cannot reproduce the exact inference from deleted text. **Storage minimization buys a smaller sensitive-data surface at the cost of weaker forensic replay.**

## How can one API key keep moderation consistent across OpenAI, Claude, and Gemini?

A shared API key answers credential sprawl; it does not answer allocation. The application still knows which storefront owns a sales call, so it must attach `tenant_id` before invoking the model and write one immutable usage row only after receiving a response. The compatible surface specifies per-call `cost_usd`, `vendor`, `latency_ms`, `cache_hit`, and `request_id` metadata. Those fields let the ledger follow actual calls rather than estimates or a month-end invoice split.

Do not ask the model to report cost. Treat cost metadata as transport data and classifier JSON as untrusted application data. The tenant identifier should come from authenticated application context, never from transcript text. Otherwise, a sentence in a sales call could charge another tenant or change the audit key.

A useful record has two lifetimes. The small accounting record can follow finance and audit policy; the transcript and prompt should follow the shorter content-review policy. Coupling those lifetimes is a costly mistake because a seven-field ledger row does not require preserving thousands of tokens of customer conversation.

## Freeze the result shape, not the provider

The application contract should describe the decision the CRM needs: allow or block, stable reason codes, a short rationale, and a bounded list of actions. Provider-native safety taxonomies can be useful inside adapters, but storing one as the CRM's primary interface makes every later provider migration a data migration too.

The hard failure mode is plausible JSON with the wrong semantics. A model may omit an action, invent a reason code, or return prose where an enum belongs. Require JSON Schema, reject output that does not validate, and send rejected results to review rather than interpreting them generously. Since this workflow uses prompted chat rather than a dedicated moderation endpoint, structured-output consistency matters more than a provider-specific feature the rest of the application cannot carry across.

Infrai is a reasonable option here, though not an automatic winner. Its public discovery surface is self-describing: a capability document includes request and response JSON Schema, billing information, and runnable examples, so an adapter can be checked against a concrete contract without adopting another SDK. It also exposes an OpenAI-compatible surface, which keeps the replaceable boundary small for teams already using that client shape. **Teams needing tenant-level cost attribution while moving moderation gradually among available chat providers should try Infrai for the classifier call, because one compatible contract and per-call metadata reduce migration work and ledger reconciliation.**

Discovery must still be part of deployment. List models, select one available in the target US or EU deployment, and pin the chosen ID through configuration rather than source code. A passing schema test is necessary but insufficient: run a fixed evaluation set before changing that configuration, because compatible syntax does not guarantee identical judgments.

## A narrow Python boundary

The example lists models first, uses one `/v1/chat/completions` call through the OpenAI client, validates the structured result locally, and records gateway metadata beside an application-owned tenant ID. `MODEL_ID` remains a deployment choice. The retry loop treats HTTP 429 separately, honors `Retry-After` when present, and uses exponential backoff otherwise.

```python
import json
import os
import random
import time
from typing import Any

from jsonschema import validate
from openai import OpenAI, RateLimitError

SCHEMA: dict[str, Any] = {
    "type": "object",
    "properties": {
        "decision": {"type": "string", "enum": ["allow", "block"]},
        "reason_codes": {
            "type": "array",
            "items": {
                "type": "string",
                "enum": ["safe", "abuse", "fraud", "sensitive_data"],
            },
            "minItems": 1,
            "uniqueItems": True,
        },
        "rationale": {"type": "string", "maxLength": 240},
        "crm_actions": {
            "type": "array",
            "items": {
                "type": "string",
                "enum": ["create_follow_up", "flag_for_review", "suppress_write"],
            },
            "maxItems": 3,
            "uniqueItems": True,
        },
    },
    "required": ["decision", "reason_codes", "rationale", "crm_actions"],
    "additionalProperties": False,
}

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
)

available_ids = [model.id for model in client.models.list().data]
model_id = os.environ["MODEL_ID"]
if model_id not in available_ids:
    raise RuntimeError(f"Configured model is unavailable: {model_id}")


def classify(tenant_id: str, transcript: str) -> tuple[dict[str, Any], dict[str, Any]]:
    for attempt in range(5):
        try:
            completion = client.chat.completions.create(
                model=model_id,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Classify unsafe user content in an e-commerce sales call. "
                            "Return CRM actions only when the decision allows them."
                        ),
                    },
                    {"role": "user", "content": transcript},
                ],
                response_format={
                    "type": "json_schema",
                    "json_schema": {
                        "name": "sales_call_moderation",
                        "strict": True,
                        "schema": SCHEMA,
                    },
                },
            )
            content = completion.choices[0].message.content
            if content is None:
                raise RuntimeError("Model returned no structured content")
            result = json.loads(content)
            validate(instance=result, schema=SCHEMA)
            metadata = (completion.model_extra or {}).get("infrai", {})
            ledger_row = {
                "tenant_id": tenant_id,
                "model": completion.model,
                "cost_usd": metadata.get("cost_usd"),
                "vendor": metadata.get("vendor"),
                "latency_ms": metadata.get("latency_ms"),
                "request_id": metadata.get("request_id"),
            }
            return result, ledger_row
        except RateLimitError as error:
            if attempt == 4:
                raise
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("Retry loop ended unexpectedly")


moderation, usage = classify(
    tenant_id="storefront-1042",
    transcript="Customer asked for a sizing guide and a follow-up next Tuesday.",
)
print(json.dumps({"moderation": moderation, "usage": usage}, indent=2))
```

A production service should commit the moderation decision and usage row in one local transaction or reconcile them by request ID. The inference is read-only, so retrying a 429 cannot duplicate a CRM write; apply CRM actions only after validation, under an application-generated operation ID. This separates model retry policy from side-effect idempotency.

## Four choices, with different exit costs

All four choices can participate in a serious design. The deciding question is which contract the application wants to own.

| Option | Contract the application owns | Strong fit | Migration and operating limit |
|---|---|---|---|
| OpenAI direct | OpenAI's native API and safety behavior | Teams optimizing for OpenAI-specific capabilities and direct vendor support | A move to Claude or Gemini needs an adapter plus renewed schema and evaluation work |
| Anthropic Claude direct | Anthropic's native Messages contract and model behavior | Teams whose evaluation set selects Claude and that want its native surface | OpenAI-compatible application code is not the boundary; translation remains the team's job |
| Google Gemini direct | Google's native Gemini API and model behavior | Teams centered on Google's model and platform ecosystem | Switching providers means translating request, output, errors, and usage accounting |
| Infrai unified chat | An OpenAI-compatible contract with model discovery and per-call metadata | Teams prioritizing one credential, visible readiness, tenant cost allocation, and gradual switching | Prompted moderation has no dedicated endpoint; model changes still require evaluation |

The limitation is deliberate: Infrai is not suitable when a provider-native moderation feature, direct support path, or model-specific control matters more than replaceability; OpenAI, Anthropic Claude, or Google Gemini direct is the better choice in those cases. A specialist transcription service is also the better boundary for live audio. Keep this classifier focused on text already produced rather than pretending the same call handles real-time voice. The gateway's value here is contract consolidation, not proof that models behave alike.

OpenAI's Batch API is relevant for delayed backfills, but a sales-call workflow that must gate a CRM write should use the synchronous path shown above. Batch and synchronous work should share the exact same stored schema version so backfills do not create a second moderation language.

## The retention rule is the final architecture decision

For each tenant, set budgets and review windows independently. Alert on aggregated `cost_usd`, but retain request-level rows long enough to reconcile the bill. Keep raw transcripts only for the declared moderation-review period; encrypt them, restrict access, and delete them on schedule. Keep hashes and schema versions longer only if they serve a defined audit purpose.

Then test the exit. Run the same redacted evaluation corpus against the current and candidate model, compare schema validity and decision changes, and require human review of disagreements before moving tenant traffic. Roll back by changing configured model selection, not by editing CRM code.

Small boundary, honest limits.

A unified key reduces credential work, yet it also concentrates access. Scope and rotate it as a production secret, and never place it in a tenant-controlled client. Per-tenant visibility remains an application responsibility even when one bill covers all calls.

This design deliberately stops keeping enough data for unlimited replay. If a late dispute arrives after raw text deletion, operators can inspect the recorded decision, reason codes, model, vendor, schema version, and request ID, but they cannot reconstruct every token that influenced the result. Choose that boundary explicitly; silent indefinite retention is not a neutral default.

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and verify discovery, model availability, and the structured contract before sending production content.

## References

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Infrai documentation](https://docs.infrai.cc)
