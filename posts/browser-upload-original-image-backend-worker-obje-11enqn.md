# Browser Upload Original Image: Backend Worker, Object Storage Webhook, Node.js Queue

Short answer: let the browser upload an immutable original image, let the backend record the tenant and deletion deadline, and use an object-storage notification only to enqueue an idempotent Node.js thumbnail job; the queue is the work ledger, while the database remains the authority for isolation, retention, and readiness.

The same boundary works for signed documents that need an explicit deletion deadline. A thumbnail is derived data. A signed document is not. That distinction matters because a retention sweep must never infer permission or legal ownership from an output key. Store those facts beside the original in a tenant-scoped database record, and make every worker operation carry that record's tenant identity.

## Who owns data governance for a tenant's original image and deletion deadline?

The pipeline has four durable states: an upload intent, an original object, a processing job, and a retention decision. An upload intent is created before the browser receives permission to write. It names the tenant, an application asset ID, the permitted media type, the expected object key, and the deletion deadline. The browser then uploads the original image to that exact key. It does not choose a tenant prefix from a form field.

Use an immutable layout such as `tenants/{tenant_id}/assets/{asset_id}/original` and `tenants/{tenant_id}/assets/{asset_id}/thumbs/{variant}.webp`. The database maps the asset ID to the key and records the current processing revision. A path is useful for listing and lifecycle work; it is not an authorization check.

The invariants are plain:

- A worker may read or write an object only after the database claim has established the tenant and asset relationship.
- A repeated storage notification must resolve to the same job ID and output keys.
- A thumbnail is not ready until all expected writes and the database transition have committed.
- The deletion deadline applies to the original and every derivative, including failed or temporary outputs.
- A request for tenant A must never accept an object key, job payload, or callback URL belonging to tenant B.

Ack last.

The dangerous gaps are easy to name. If the receiver acknowledges a notification before the queue accepts the job, a process exit loses work. If the worker acknowledges before recording the output keys, a retry cannot tell whether it should rewrite or wait. If cleanup deletes the database row first, the retention process loses the metadata needed to find all derived objects. Each ordering choice creates a recovery obligation, so document it as part of the decision record rather than hiding it in a helper function.

## How should a browser upload integrate with object storage?

Yes, but each component must have one job. The browser owns the original-image transfer and upload progress. Object storage owns bytes and emits a notification. The backend receiver validates that notification and inserts a small queue message. A Node.js worker decodes the original, generates thumbnails, and commits their state. The retention worker deletes objects only after checking the same tenant-scoped record.

Do not put image bytes in the notification or queue. Send an asset ID, tenant ID, original key, and processing revision. The receiver should derive the job ID from stable values, then perform a uniqueness check in the database before enqueueing. At-least-once delivery is acceptable when the consumer is idempotent; exactly-once notification delivery is not a prerequisite for correctness.

The browser can show transfer progress using XMLHttpRequest upload progress events, as documented by MDN. That progress ends when the original request ends. It says nothing about decoding, resizing, queue delay, or retention. Expose those as application states instead: `uploading`, `uploaded`, `processing`, `ready`, and `deleting`. A UI that labels a completed upload as a ready thumbnail is reporting the wrong fact.

Here is the critical path in Python, with the storage and database contracts left explicit. The production worker may run in Node.js; the example is intentionally about ordering and ownership, not about a particular SDK. The `claim` operation must be an atomic database transition, and `mark_ready` must commit the complete variant map.

```python
from dataclasses import dataclass
from hashlib import sha256


@dataclass(frozen=True)
class ThumbnailJob:
    tenant_id: str
    asset_id: str
    original_key: str
    revision: int

    @property
    def job_id(self) -> str:
        raw = f"{self.tenant_id}:{self.asset_id}:{self.revision}"
        return sha256(raw.encode("utf-8")).hexdigest()


VARIANTS = {"small": 320, "detail": 1280}


def process(job, storage, database, resize_to_webp):
    if not database.claim(job.job_id, job.tenant_id, job.asset_id, job.revision):
        return "already-claimed-or-complete"

    try:
        original = storage.get_for_tenant(job.tenant_id, job.original_key)
        outputs = {}

        for name, max_edge in VARIANTS.items():
            data = resize_to_webp(original, max_edge=max_edge)
            key = (
                f"tenants/{job.tenant_id}/assets/"
                f"{job.asset_id}/thumbs/{job.revision}/{name}.webp"
            )
            storage.put_for_tenant(
                job.tenant_id,
                key,
                data,
                content_type="image/webp",
                idempotency_key=f"{job.job_id}:{name}",
            )
            outputs[name] = key

        database.mark_ready(job.job_id, job.tenant_id, outputs)
        return "ready"
    except Exception:
        database.release_for_retry(job.job_id)
        raise
```

The tenant argument is deliberately repeated at the storage boundary. It may look redundant beside a tenant-prefixed key, but redundancy is useful when a key is malformed, copied between environments, or produced by an old client. The adapter should verify that the key belongs to the claimed tenant before performing the operation. A prefix is a namespace convention; the authorization decision belongs in code and policy.

## Can a backend Node.js worker generate thumbnails from a storage webhook?

Start with the failure that is hardest to repair: cross-tenant exposure. Never trust a browser-supplied object key or callback destination. Resolve the asset ID server-side, bind it to the authenticated tenant, and reject a notification whose bucket, key, or metadata does not match the pending intent. Log tenant ID and asset ID as structured fields, but do not log signed URLs or document contents.

The next failure is duplicate work. A storage notification can be repeated, and a queue message can become visible again after a worker has committed but before its acknowledgement reaches the queue. The job ID, revision, deterministic output keys, and atomic claim turn that repeat into a harmless lookup. This is why the database state must include `ready` and `deleted`, not just a boolean called `processed`; deletion and reprocessing are different transitions.

There is a longer example worth testing. Tenant `shop-17` uploads asset `a-204` at revision 3. Two notifications arrive. The first inserts the job; the second finds the uniqueness constraint. A worker claims the row and writes `small.webp`, then the process stops before `detail.webp`. A retry sees an incomplete revision, writes both deterministic keys, and commits one complete output map. If the acknowledgement is lost after that commit, another delivery sees `ready` and exits without decoding the image. Now advance the clock past the deletion deadline: the retention worker claims the asset, deletes the original and both derivatives, then marks the record `deleted`. If it stops after the original delete, the next run uses the stored output map and continues. Nothing depends on an event arriving exactly once.

That sequence is the test. I would also inject a notification with tenant `shop-18` and the key for `shop-17`; the receiver must reject it before enqueueing. A useful negative test is a thumbnail key with the right asset ID but the wrong tenant prefix. It must fail closed. Three words: isolate first.

Retention has its own race. A late thumbnail job must not recreate data after the deadline. The worker should check the current record before each write, and the database claim should be invalidated by a deletion transition. The storage lifecycle rule is a second line of defense, not a substitute for application state, because derived variants and temporary objects may not share identical prefixes or deadlines.

## How do you test and prove the deletion deadline?

The useful comparison is about control and failure boundaries, not a generic feature count.

| Choice | What it gives the pipeline | Cost or limit | Use it when |
|---|---|---|---|
| Direct browser-to-object upload | Less application bandwidth and visible upload progress | Requires careful authorization, expiry, and cross-origin policy | The storage boundary can enforce the intended upload contract |
| Backend-proxied upload | One place to inspect bytes and enforce policy | More backend bandwidth and request latency | Tenant isolation or content inspection outweighs transfer efficiency |
| Storage notification | A low-latency signal that an object event occurred | It is not a durable work ledger and may repeat | The receiver can validate and enqueue durably |
| Queue | Retry, backoff, concurrency control, and acknowledgement | Adds another durable component and operational state | Thumbnail work may outlive the upload request |
| Database record | Tenant binding, revision, readiness, and deletion authority | Must be kept consistent with object writes | The application needs auditability and reconciliation |
| Lifecycle policy | Automatic expiry for a known object prefix | Cannot express every tenant-specific processing state | It supplements an application retention worker |

The rejected option is synchronous resizing inside the browser upload request. It couples a user-visible transfer to image decoding and multiple derived writes, and it makes a disconnected client an ambiguous coordinator. It is valid for a tiny, single-output tool where the derivative is disposable and no explicit tenant retention policy exists. It is not suitable for e-commerce records with signed documents, tenant isolation, or an auditable deletion deadline.

The catch is that a queue does not make bad state disappear. It adds a place to retry bad state. If the team cannot operate a database, a queue, and reconciliation metrics, a backend-proxied upload followed by a transactional outbox may be a better operational fit, even though it uses more application bandwidth. Your mileage will vary with object volume, worker runtime, and regulatory retention rules; those inputs should be written into the ADR before anyone compares storage prices.

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
