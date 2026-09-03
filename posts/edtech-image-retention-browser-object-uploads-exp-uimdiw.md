# Edtech Image Retention: Browser Object Uploads, Express Signing, Private Reads

Short answer: for an edtech product that stores learner and course images, keep the bucket private, let Express authorize a short-lived presigned PUT for a browser upload, and issue a separate signed download URL only after the tenant and retention checks pass. The important design decision is not where the bytes travel. It is who can delete them, when deletion becomes irreversible, and how the application proves that a tenant-scoped export includes the right objects.

That distinction matters because a direct upload moves data around the application server without moving responsibility for it. A React client can send bytes to object storage, but it cannot decide that an object belongs to tenant A, that a retention period has expired, or that a download is allowed. Those decisions stay in the Express control plane and its database.

## Start with a deletion ledger, not a storage endpoint

For an edtech product, the first record should describe the obligation to retain or remove an image. Store the tenant, object key, retention class, deletion request time, export references, and the actor who changed the state. A bucket can hold bytes; it cannot explain why an image was still downloadable after a course closed or which export included it.

Use an application state such as `pending`, `ready`, `deletion_pending`, and `deleted`. A deletion request should stop new signed reads immediately, enqueue object deletion, and only present a completed result after the worker has confirmed the intended operation. Keep an audit event for both the request and the result. The exact confirmation mechanism depends on the storage system, but the user-facing claim should never be stronger than the evidence behind it.

Exports need their own retention class. Deleting a source row does not remove a ZIP that was generated last week, and deleting the object first can make a still-valid export incomplete. This ledger is the decision axis: it turns direct upload from a convenience feature into a controlled data lifecycle.

It is a ledger, not a folder.

## How should an edtech export use object keys and signed downloads?

The worker should create a manifest from authorized application rows, not from a bucket prefix. Each row can record the tenant, opaque object key, expected size or checksum when available, and retention state. The worker then reads only rows that are still exportable, records omissions rather than silently widening the query, and publishes the artifact only after the manifest and output have been finalized.

That makes a tenant-scoped export a point-in-time statement rather than a lucky listing. Decide explicitly whether the artifact is a snapshot or a live collection. Give it an independent key, expiry, and application record; otherwise, a successful image deletion can coexist with an old downloadable copy.

Short lists are useful here. They expose the state transitions that a feature checklist tends to hide.

## What should a React browser upload prove before a private bucket read?

The upload should create an application record before it creates a storage capability. The record needs an opaque object key, tenant identifier, intended content type, original filename, uploader, retention class, and a state such as `pending`, `ready`, or `deleted`. Express chooses the key; the browser should not be allowed to choose a path that can collide with another tenant's data.

The sequence is deliberately unglamorous:

1. React asks an authenticated Express endpoint to start an upload.
2. Express checks tenant membership, content-type policy, and any product-specific retention rule, then reserves an object key.
3. Express returns a presigned PUT URL and the exact headers the browser must send.
4. React uploads the bytes directly and reports completion to Express.
5. Express verifies object metadata, changes the record to `ready`, and later authorizes a signed download URL for a particular tenant and user.

Issuing the URL is not completion. A client can abandon a page after receiving it, retry a request, or report success for a key it did not upload. Completion should therefore be idempotent. Repeating the same completion request must converge on one database state, while a completion for an unknown or differently owned key must be rejected by the application.

A private bucket is useful only if every read follows the same authorization path. Do not put a permanent object address in a course page and assume obscurity is access control. A signed download URL is a temporary capability, so its lifetime should reflect the actual viewing workflow; the database remains the source of truth for tenant ownership and deletion.

The browser still enforces its cross-origin rules. A command-line PUT can succeed while a React page fails during the `OPTIONS` preflight because the storage CORS policy does not allow the deployed origin, the `PUT` method, or a requested header. The signature answers an authorization question. CORS answers whether the browser may make the request. They are separate checks.

This is a common 403-shaped trap: a developer sees a valid-looking URL, tests it outside the browser, and blames the signer when the page's origin was never permitted. Test from the real staging origin, inspect both preflight and upload responses, and keep the signed request's method, content type, and headers identical to the values used when the URL was created. MDN's CORS documentation is the right baseline for understanding which response headers the browser requires.

The same boundary affects tenant-scoped exports. The export worker should read authorized object keys from application data, stream or copy only those keys into an export area, and create a separate download capability for the finished artifact. Listing a prefix is not a substitute for a tenant query: prefixes are naming conventions, not an ownership proof.

Retention adds a harder constraint. “Delete the row” and “delete the object” are different operations, and an export can outlive the image records that produced it unless it has its own retention class. Define the state transition explicitly. For example, a request can mark an image `deletion_pending`, prevent new signed reads, enqueue object deletion, and only then mark the application record `deleted` after the worker has confirmed the intended operation. The exact confirmation mechanism depends on the storage system, but the invariant should not: a user-facing deletion result must not claim more than the system has established.

Treat each transfer as a small state machine rather than a single HTTP request. An expired PUT URL is an expected lifecycle event; the client can ask Express for a fresh capability after the application confirms that the pending record is still valid. A failed download authorization should not be “fixed” by making the bucket public.

For uploads, use a stable application upload ID and a newly generated object key. Store no bearer credential in React. Never log the full presigned URL: query parameters can grant temporary access, and logs often survive the page that produced them. Log the upload ID, tenant, key hash, outcome, and request correlation ID instead.

I treat a 413 response as a policy signal, not as a retryable network failure. The client should surface the size limit and stop. A timeout or a transient rate limit needs a different policy, but retrying a PUT blindly can create duplicate objects unless the key and completion record are stable. I'm not sure a universal retry delay is possible here; storage limits, browser behavior, and the chosen transfer mode determine the useful backoff, so verify it with the actual files and origins used by the product.

Deletion needs the same discipline. An expired lifecycle rule can remove abandoned uploads, but application deletion still needs an authorization record and an audit event. A lifecycle policy is a storage safety net, not a tenant policy engine. The AWS object lifecycle guidance is a useful reference for the distinction between lifecycle configuration and object management, and its operational details should be checked against the storage implementation being evaluated.

## What should the signing boundary return to React?

The control plane should return a capability and a narrow contract, not storage credentials. This Python example is intentionally provider-neutral: the `sign_put` and `sign_get` functions stand for the storage adapter, while the tenant and lifecycle checks remain application code. A Node.js Express service can expose the same two operations without changing the browser contract.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class UploadIntent:
    upload_id: str
    tenant_id: str
    object_key: str
    put_url: str
    put_headers: dict[str, str]
    expires_at: datetime


def create_upload_intent(user, tenant_id, filename, content_type, store, records):
    if not user.can_upload_to(tenant_id):
        raise PermissionError("tenant access denied")
    if content_type not in {"image/jpeg", "image/png", "image/webp"}:
        raise ValueError("content type denied")

    upload_id = records.reserve_upload(
        tenant_id=tenant_id,
        filename=filename,
        content_type=content_type,
        retention_class="course-image",
    )
    object_key = records.opaque_key(upload_id)
    expiry = datetime.now(timezone.utc) + timedelta(minutes=10)
    put_url = store.sign_put(object_key, content_type=content_type, expires_at=expiry)
    records.attach_key(upload_id, object_key)
    return UploadIntent(
        upload_id=upload_id,
        tenant_id=tenant_id,
        object_key=object_key,
        put_url=put_url,
        put_headers={"Content-Type": content_type},
        expires_at=expiry,
    )


def authorize_download(user, upload_id, records, store):
    item = records.get(upload_id)
    if item is None or item.state != "ready":
        raise LookupError("object is not available")
    if not user.can_read_tenant(item.tenant_id):
        raise PermissionError("tenant access denied")
    return store.sign_get(item.object_key, expires_in=300)
```

The browser sends the returned `Content-Type` exactly as signed, then calls a completion endpoint with `upload_id`; it does not send a platform authorization header to the object URL. The completion handler should inspect the object, verify that the reserved record is still pending, and atomically move it to `ready`. The download function checks the tenant again, because access can change after upload.

## Put the data plane behind a narrow control plane

Keep the control plane and data plane separate. Express owns authorization, object intent, retention metadata, and export jobs. The browser owns neither storage credentials nor deletion decisions. A worker owns long-running export and cleanup work, with a queue that can retry idempotently. Object storage owns byte durability and temporary transfer capabilities.

The failure modes worth testing are concrete:

| Failure mode | Observable symptom | Design response |
|---|---|---|
| CORS preflight denied | Browser reports a cross-origin failure before PUT | Test the deployed origin and requested headers, not only the URL outside a browser |
| URL expires mid-flow | PUT or download is rejected after waiting | Re-authorize through Express and issue a new short-lived capability |
| Duplicate completion | Two callbacks create conflicting records | Use an upload ID and an idempotent state transition |
| Cross-tenant key guess | A user can name another tenant's object | Generate opaque keys and authorize every key through application data |
| Export outlives deletion | A deleted image remains in an old artifact | Give exports their own retention and revocation policy |
| Abandoned upload | Pending bytes accumulate | Apply a documented cleanup rule and reconcile it with application state |

## Roll out the boundary with failure evidence

Start with invariants, not a provider logo: private-by-default reads, tenant ownership outside object names, a defined deletion acknowledgment, a recoverable export job, browser CORS control, and an answer for abandoned uploads. Then test the exact control plane and data plane together.

The catch is that direct browser transfer is not suitable when the product must inspect or transform every byte synchronously, when browser CORS cannot be configured to match the deployed origins, or when immutable retention and legal hold are hard requirements that the selected storage boundary cannot provide. In those cases, use a server-mediated upload or choose a storage system whose documented controls satisfy the requirement. Stick with direct transfer when the object can be validated by metadata and asynchronous processing, and when keeping large request bodies out of Express is the real operational need.

Roll out with one disposable tenant and one short-lived test object. Verify preflight, content-type binding, URL expiry, duplicate completion, tenant filtering, export manifest contents, deletion acknowledgment, and cleanup of abandoned records. Add metrics for pending age, signed URL issuance, PUT completion, export omissions, and deletion lag. A green upload button proves very little; the useful test is whether the system can explain, later, why a particular tenant could read or could no longer read a particular object.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
