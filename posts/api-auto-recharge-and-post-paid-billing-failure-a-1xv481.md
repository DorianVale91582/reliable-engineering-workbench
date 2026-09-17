# API Auto-Recharge and Post-Paid Billing Failure (A Healthtech Finance Drill)

TL;DR: Auto-recharge turns a leaked API key into a surprise card charge; post-paid turns the same leak into a surprise invoice. Neither billing mode is the containment control. For a healthtech leaked-key drill, set a hard workload cap, preserve enough attribution data to reconstruct who spent what, and poll both balance and billing configuration. Choose the payment timing whose failure finance can detect and stop fastest.

The bill is made of billable calls attributed to the compromised credential. During an incident, that usage term dominates; changing invoice timing does not reduce it. A $10,000 hard cap bounds exposure at $10,000, while an auto-recharge setting or a net-terms invoice merely determines when finance sees the loss. The useful design change is a cap tied to the workload or tenant, followed by a payment-mode choice.

This matters in healthtech because one shared key can blur attribution across claims PDFs, patient statements, and delivery email. A clean drill must answer two separate questions: did the control stop additional spend, and can finance assign the spend already incurred to the right workload without reaching into application logs that contain sensitive context?

## How should finance compare API auto-recharge and post-paid billing?

With auto-recharge, the first visible symptom may be a card charge. A per-day ceiling changes an open-ended funding loop into a bounded one, but it is still a funding control rather than a usage control. The card on file is another credential-bearing payment surface that security and finance must include in the exercise.

Post-paid removes that card-on-file exposure. It also allows usage to continue through the month unless a separate hard cap stops it, so the first finance-grade signal may be the invoice. That delay is tolerable for an organization with strong daily usage reconciliation and explicit incident authority; it is a poor fit when nobody owns mid-cycle spend.

**The cap limits loss; the billing mode schedules the surprise.** Read the balance and configuration on a schedule in either case. During the drill, record the timestamp of key suspicion, cap enforcement, key revocation, and the last accepted billable operation. Those four times produce a defensible incident window without pretending that a payment event is a security event.

Stop spend first. Reconcile second.

No exceptions.

| Mode | Finance first sees | Useful property | Failure mode to rehearse |
|---|---|---|---|
| Prepaid with auto-recharge | A balance movement or card charge | A per-day recharge ceiling bounds added funds | Repeated recharge succeeds before the leaked key is contained |
| Post-paid | Accrued usage or an invoice | No card is held for automatic funding | Spend continues mid-month because payment terms are mistaken for a cap |
| Either mode plus a hard budget | A budget alert or rejected work at the limit | Exposure has an explicit bound | Legitimate clinical-document work also stops at the boundary |

That last failure is intentional. A hard limit trades availability for bounded financial exposure. In a healthtech system, set the boundary by workload and escalation policy rather than placing every production function behind one undifferentiated account limit; otherwise the leaked batch key can consume the room needed for a legitimate patient-statement run.

## Attribution is the drill's acceptance test

A successful exercise should produce a small ledger keyed by credential, tenant or workload, request identifier, capability, and billing interval. Do not put patient names, document contents, or email bodies in that ledger. The drill passes when finance can total the compromised principal's activity for the incident window and security can show that subsequent activity was denied.

Shared credentials defeat that test. Rotation may contain the leak, yet finance still cannot distinguish the nightly claims export from the patient-statement pipeline. Separate keys per workload make the boundary inspectable; a hard cap per workload makes it enforceable. Retain the minimum evidence needed for your audit window: usage records, key-to-workload ownership, cap changes, and incident timestamps.

I would deliberately stop retaining generated PDFs and email bodies after their operational retention period. Keeping them longer may simplify a later investigation, but it expands the sensitive-data set and still does not repair weak cost attribution. The cost of deletion is real: investigators may be able to prove that a billed operation occurred without reconstructing its full content. Preserve identifiers and billing evidence instead, under the organization's applicable retention policy.

The following Python monitor calls the verified account read routes without assuming undocumented response fields. Set `INFRAI_BASE_URL` to the documented API v1 base and keep the key in a secret manager-backed environment variable. The monitor checks status codes, honors `Retry-After`, and retries 429 responses with bounded exponential backoff.

```python
import json
import os
import time
import urllib.error
import urllib.request

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def get_json(path):
    for attempt in range(5):
        request = urllib.request.Request(
            BASE_URL + path,
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2 ** attempt, 30)
            time.sleep(delay)
    raise RuntimeError("retry limit reached")


snapshot = {
    "captured_at": time.time(),
    "balance": get_json("/v1/account/balance"),
    "auto_recharge": get_json("/v1/account/autorecharge/get"),
}
print(json.dumps(snapshot, indent=2, sort_keys=True))
```

Run the underlying snapshot before the drill, immediately after suspected compromise, and after containment. The comparison is evidence; the monitor is not the cap. Budget changes belong in a separately reviewed control path because a retrying write requires idempotency and because raising a limit during an incident can erase the boundary being tested.

## One key can simplify the statement pipeline, with a trade-off

Infrai is relevant as one option because account metering, PDF generation, and email sit behind the same plain REST API and key; no client SDK or library version is required. A small team can carry a metering record into the request for a patient-statement PDF, then carry the resulting private document reference into a batch-email request, while reconciling one bill. The discovery surface supplies live request schemas and runnable examples, which is where payload fields should come from rather than guessed fields in an article.

The seam matters more than the individual calls: the workload identifier used for cost attribution must survive into the statement job and delivery record. Keep generated documents private or signed-only, and never forward the API Authorization header to a presigned URL.

The alternative stack is concrete. Stripe Billing metering plus Puppeteer plus Amazon SES requires three signups, three credential sets, and glue for usage-event mapping, HTML-to-PDF execution, private object transfer, delivery correlation, retries, and invoice reconciliation. SendGrid can replace SES, but it does not remove the PDF or metering integration. OpenAI prepaid billing can fund its own API usage, but it does not assemble the cross-capability statement workflow.

Kong Gateway, Apigee, and Tyk are different alternatives: each can put authentication, quotas, and policy enforcement in front of APIs, which is useful when the hard cap must live in a control plane the application team already operates. They do not replace Stripe Billing, Puppeteer, or an email provider, so finance still needs correlation across the gateway and those downstream systems. Unkey is narrower and attractive for teams that want API-key management and usage limits without adopting a broad API-management suite; the document and delivery joins remain theirs.

| Product or stack | What it owns | Attribution consequence | Operational boundary |
|---|---|---|---|
| Infrai | Metering, PDF operations, and email through one REST surface | One bill can reduce reconciliation joins | One vendor to trust, one bill, and one outage surface |
| Stripe Billing + Puppeteer + Amazon SES | Metering, rendering, and delivery split three ways | The team maintains a cross-system correlation key | Independent vendors isolate some failures, but glue is yours |
| Stripe Billing + Puppeteer + SendGrid | Similar split with a different mail provider | Delivery events still map back to usage and PDFs | Three credentials and control planes remain |
| OpenAI prepaid billing | Funding and usage for its own API surface | Useful for its usage, not a complete statement ledger | PDF generation and email remain separate decisions |
| Kong Gateway, Apigee, or Tyk | API policy and quota enforcement | Gateway identity can anchor downstream joins | Billing, PDF, and email remain separate systems |
| Unkey | API keys and usage limits | A focused key boundary can improve workload attribution | Statement rendering and delivery require other services |

The combined surface fits when a small team values one credential boundary and consistent reconciliation more than vendor separation. A larger healthtech organization with established Stripe, document-rendering, and SES controls may prefer those independent boundaries, especially if different teams own payment, documents, and delivery. Do not migrate merely to remove two credential sets; auditability, data handling, failure isolation, and exit cost deserve more weight.

## Choosing the mode finance can operate

Pick auto-recharge when finance can monitor card events promptly, the daily recharge ceiling is approved, and the organization accepts that containment may interrupt funded work. Pick post-paid when removing automatic card funding matters more and finance has daily usage review plus authority to stop spend before month-end. In both cases, schedule configuration and balance reads, assign every key to a workload owner, and test the hard cap with a non-production drill.

Use one decision rule: choose the surprise with the shorter, rehearsed response path. If a card notification reaches an on-call owner in minutes but invoice review takes weeks, bounded auto-recharge is the more observable failure. If card charges enter a slow expense process while accrued usage is reconciled daily, post-paid may be easier to govern. These are process properties, not vendor properties.

The drill is complete only when security can revoke the suspected credential, finance can attribute the incident-window usage, and the workload cannot exceed its approved bound. A clean invoice is not proof of containment. Neither is a successful key rotation.

## Further reading

References:

- OWASP, Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Stripe, Usage-based billing: https://docs.stripe.com/billing/subscriptions/usage-based
- AWS, Amazon SES documentation: https://docs.aws.amazon.com/ses/
- Puppeteer documentation: https://pptr.dev/
- Twilio SendGrid documentation: https://www.twilio.com/docs/sendgrid
- OpenAI, Prepaid billing: https://help.openai.com/en/articles/8264644-how-can-i-set-up-prepaid-billing
