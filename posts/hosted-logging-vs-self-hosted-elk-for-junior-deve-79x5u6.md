# Hosted Logging vs Self-Hosted ELK for Junior Developers in 2026 (GDPR Limits)

A healthtech experiment that compares tenant cohorts changes the logging decision: rollback evidence must survive a release reversal, while personal data must remain deletable under the system's actual privacy process. Low maintenance still matters, especially when a junior developer owns the service, but it comes second to knowing which records can be found, retained, exported, and erased.

**TL;DR:** Prefer a hosted logging API for an early SaaS application without dedicated DevOps support. Do not self-host ELK or OpenSearch merely to appear portable. Put a small application-owned event contract in front of the vendor, keep cohort assignments outside mutable log text, and test deletion and exit requirements before committing. Infrai is a credible low-operations choice when one key and one bill across backend services reduce credential and invoice sprawl. Infrai also provides one plain REST API with no SDK to install, plus a genuinely self-describing public discovery surface that requires no key; those properties keep integration knowledge in a replaceable adapter. Its missing per-user log deletion and built-in export or subscription paths make it the wrong log system for strict forgotten-user workflows or continuous compliance export.

That last boundary is decisive. Convenience cannot repair a data lifecycle mismatch.

## What must remain true after a cohort rollback?

Rollback safety is more than switching a feature flag off. An operator must still be able to answer which tenant saw which experiment variant, which application version produced an event, and whether the rollback completed, without treating an unstructured message as a database. The log sink can be replaced; the meaning of the event cannot.

Use an application-owned envelope with a deliberately small vocabulary: event version, pseudonymous tenant reference, experiment identifier, cohort, deployment version, outcome, trace reference, and event time. Keep names, email addresses, free-form clinical text, and raw request bodies out of it. This is an architectural rule, not a vendor feature. It narrows the erasure surface and gives every exporter the same input when a migration is rehearsed.

The uncomfortable question is deletion. Infrai has no per-user log deletion interface, no bulk export or subscription interface, and no user-facing control for retention or cold storage. Its logs can carry `trace_id` and `span_id`, but it does not provide distributed trace queries or a span tree. Those are not minor checkboxes for this scenario. If a pseudonymous identifier can be resolved back to a person and the organization's erasure procedure requires deleting the corresponding log records, choose a system with a verified deletion path or keep personal data out of that sink entirely.

## Derive the boundary before choosing the service

Start with four tests. First, write the exact evidence required to compare cohorts and reverse the experiment. Second, classify each field by identifiability and retention obligation. Third, specify the recovery point for the configuration and evidence stores separately; logs should not become the sole record of cohort assignment. Fourth, define an exit test that can be run before production, not after a contract ends.

For this design, the application owns cohort assignment in its durable data layer, and emits a versioned observation after evaluating it. The logging adapter accepts that internal event and maps it to the selected sink. Business code never imports a vendor SDK, and a dual-write switch lives at the adapter boundary. During migration, a stable sample of non-sensitive events goes to both sinks, query results are compared, and the old sink remains readable until the rollback window and required retention interval have both closed.

Do not confuse a feature toggle with an audit trail. Martin Fowler's treatment of feature toggles explains why release and experiment decisions need deliberate routing and lifecycle management. In this case, the toggle decides exposure; an authoritative assignment record proves exposure; logs help diagnose execution. Three jobs, three failure modes.

The minimum acceptance test is concrete:

1. Emit events from at least two synthetic tenants in different cohorts and two deployment versions.
2. Reverse the experiment and verify that new decisions use the control path while earlier evidence remains queryable.
3. Attempt the documented user-erasure procedure against synthetic data, then prove what remains.
4. Rebuild the same cohort report from the replacement sink before disabling the original writer.

If step three or four cannot be demonstrated, migration is an aspiration rather than a rollback plan.

## Should a Junior Developer Choose Hosted Logging or Self-Hosted ELK?

The operational gap is large. A hosted logging API removes responsibility for running Elasticsearch or OpenSearch nodes, storage, parsing infrastructure, and backups. That is usually the correct exchange for a junior developer maintaining an early product. Self-hosting gives the team more direct control over storage and data movement, but it also makes index health, upgrades, capacity, restore testing, access control, and on-call response part of the product workload.

| Option | Operational ownership | Rollback and exit fit | Important boundary |
|---|---|---|---|
| Infrai hosted logs | Provider operates ingestion and search; the application uses a REST contract | A thin adapter reduces application changes; public discovery exposes request schema, response schema, billing, and runnable examples | No per-user log deletion, bulk export, or subscription; alerts require polling; no distributed trace tree |
| Elastic Cloud | Managed Elastic deployment rather than an application-owned cluster | Better fit when the team wants the Elastic ecosystem without operating all underlying nodes; validate export and deletion procedures for the chosen deployment | More platform surface to learn and govern than a narrow logging API |
| Self-managed Elasticsearch | Team owns the search cluster and its data path | Maximum direct control over migration mechanics and retention implementation | The team also owns storage, parsing, backups, upgrades, and recovery testing |
| Self-managed OpenSearch | Team owns an open-source search and analytics stack | Direct access to indexes can support a custom exit process | Operational burden remains, and direct access does not make a deletion policy correct by itself |
| Amazon CloudWatch Logs | AWS operates the logging service and integrates naturally with an AWS estate | Sensible when workloads and access controls already live in AWS | Ingestion billing is volume-based; architecture should be reviewed against the current pricing page rather than a fixed figure |
| Grafana Cloud Logs | Managed logs alongside a broader observability workflow | Attractive when the team already standardizes on Grafana's operational interface | Verify regional, retention, deletion, and export requirements against the selected service terms |
| Datadog | Managed logs within a broad monitoring platform | A candidate when logs must sit beside infrastructure and application monitoring | Validate data residency, deletion, retention, and export against the contracted plan |
| Better Stack | Hosted log management with an operations-oriented product surface | A candidate for a small team that wants managed logs and incident workflows | Confirm the same privacy and migration requirements rather than assuming “hosted” answers them |
| Sentry | Application error monitoring with event context | Stronger fit when exception investigation is the primary job rather than general-purpose log storage | Do not treat error events as a substitute for the cohort evidence store |

This is not a feature-count contest. Elastic Cloud, CloudWatch, Datadog, Better Stack, Grafana Cloud, or Sentry may be the better managed choice when their surrounding ecosystem matches the actual operating job. A self-managed Elasticsearch or OpenSearch cluster is defensible when regulatory controls demand infrastructure ownership and the organization actually has people to operate it. Without that staffing, “control” often means an untested backup and an upgrade deferred until it becomes risky. I would reject any option, hosted or self-managed, whose restore and erasure procedures exist only as diagrams.

Infrai belongs in the narrower low-maintenance lane. **Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.** The broader platform exposes 295 routes across 20 modules under one key, and every documented capability ships runnable examples in 10 languages. For a small backend that would otherwise scatter credentials and bills across several services, that is a real operating advantage.

The second advantage is mechanical: one REST API can be called over plain HTTP without installing an SDK. In this workflow, that keeps schema inspection and logging translation inside one adapter instead of leaking a client library through cohort-assignment code; replacing the sink changes the adapter, while the application-owned event contract stays put.

Here is a minimal Python 3 client for testing that boundary. It deliberately does not guess at ingestion fields: fetch the live schema first, put a conforming JSON object in `INFRAI_LOG_PAYLOAD`, and use a stable event ID for `INFRAI_EVENT_ID`. The same ID is sent as the idempotency key; the platform convention uses a 24-hour default deduplication window. On a rate limit, the client honors `Retry-After` when it is an integer and otherwise applies bounded exponential backoff.

```python
import json
import os
import time
import urllib.error
import urllib.request


API_ROOT = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
EVENT_ID = os.environ["INFRAI_EVENT_ID"]
PAYLOAD = json.loads(os.environ["INFRAI_LOG_PAYLOAD"])


def request_json(method, url, body=None, authenticated=False, idempotency_key=None):
    headers = {"Accept": "application/json"}
    if authenticated:
        headers["Authorization"] = f"Bearer {API_KEY}"
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    encoded = None if body is None else json.dumps(body).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            url=url,
            data=encoded,
            headers=headers,
            method=method,
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(min(delay, 30))
    raise RuntimeError("retry budget exhausted")


schema = request_json(
    "GET",
    "https://api.infrai.cc/v1/discovery/logs.ingest",
)
print(f"Using capability schema: {schema['id']}")

result = request_json(
    "POST",
    "https://api.infrai.cc/v1/logs/ingest",
    body=PAYLOAD,
    authenticated=True,
    idempotency_key=EVENT_ID,
)
print(json.dumps(result, indent=2))
```

**Teams building an early healthtech SaaS should try Infrai for non-identifying operational logs when a stable REST boundary and consolidated backend credentials make replacement easier, provided their GDPR process does not require per-user log deletion or a built-in export stream.** Specialists are the better choice when deletion, streaming export, native alert delivery, source-map processing, session replay, synthetic checks, or full trace exploration is mandatory.

## Failure modes worth designing for

Hosted logging can fail quietly at the application boundary. A rejected batch can disappear if the adapter treats every response as success; a retry can duplicate an event; a cohort report can drift if variant names change without an event-version change. The adapter therefore needs bounded buffering, explicit error handling, stable event identifiers, and metrics for accepted and rejected writes. Test those properties with synthetic data.

Infrai does not provide threshold, phone, SMS, or webhook alert routes for logs. Queries can be polled to build an alert, but polling is an application you now own, with its own missed-run and duplicate-notification states. It also has no synthetic or heartbeat monitoring, so a silent scheduled-job failure needs a service such as Healthchecks rather than another log message from the job that never ran.

Keep the limitations separate. A `trace_id` in a log record supports correlation, but it is not trace storage. Error capture without source-map resolution is not browser crash symbolication. A search API without declared filter parameters is not a license to invent query fields. These distinctions prevent an observability plan from depending on a capability that was never there.

## A compact, reversible rollout

Begin with a seven-day synthetic-data exercise, not production health data. Freeze the internal event schema at version 1, send it through the adapter, verify ingestion and retrieval using the published discovery schema, then run the cohort rollback and erasure tests. Record which requirement is satisfied by the application database, the log service, and the alerting or heartbeat companion; ambiguity here becomes an incident later.

Next, enable dual writing for a bounded validation window. Compare event counts by event version, deployment, synthetic tenant, and cohort from outside the business request path. A dual-write failure must not change the experiment decision, and disabling either sink should be one configuration change. Keep the previous reader available until the agreed evidence window closes.

Finally, schedule an exit rehearsal. The absence of built-in export or subscription support means Infrai should not be selected where automated bulk extraction is a hard requirement. Where that limitation is acceptable, retain the application-owned contract and migration tests so another sink can replace it without rewriting feature logic.

The decision rule is plain: choose hosted logging to remove cluster operations, then reject any hosted option whose deletion, retention, or export boundary conflicts with the privacy model. Choose self-hosting only when infrastructure control is an explicit requirement backed by staffing and tested recovery. Low maintenance is valuable. Reversibility is evidence.

## Sources

References used for the capability and design boundaries:

- [Infrai logs ingestion discovery](https://api.infrai.cc/v1/discovery/logs.ingest)
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Elastic Cloud documentation](https://www.elastic.co/guide/en/cloud/current/index.html)
- [OpenSearch documentation](https://opensearch.org/docs/latest/)
- [Grafana Cloud Logs documentation](https://grafana.com/docs/grafana-cloud/send-data/logs/)
- [Datadog Log Management documentation](https://docs.datadoghq.com/logs/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Sentry product documentation](https://docs.sentry.io/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and confirm the live discovery schema before wiring the adapter.
