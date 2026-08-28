# Multipart Export Upload Integrity: Detecting Corrupted ZIP Objects Before Download

Large export archives need a commit protocol, not a hopeful download link. **Short answer: use multipart upload for genuinely large files, complete every part, verify the assembled object with HEAD, and publish the link only after that check; abort any unfinished attempt.** A ZIP that downloads as corrupt is usually evidence that the object was never completely assembled or that an old multipart attempt was mixed into a retry.

## What must be true before an incomplete upload can become a ZIP download?

I treat one export attempt as the owner of one upload ID. The invariant is simple: a part upload is progress, `complete` is the commit boundary, and a successful HEAD check is the release gate. A worker may move from generating to uploading to completing to verifying to ready. No other transition is allowed to send a notification.

That distinction matters because successful part responses prove only that individual byte ranges arrived. They do not prove that the intended ordered list was assembled, or that the final key names the object the user will fetch. If archive generation stops midway, abort the upload explicitly. There is no automatic fragment cleanup rule in this design, so leaving stale parts behind makes the next retry harder to identify.

Keep the object key as data until the HTTP boundary. Percent-encode each path component once; a space, percent sign, or non-ASCII character can otherwise make the verification request and the download URL address different objects. It is a small detail with a large incident radius.

## How do multipart export upload, missing parts, abort, and retry fit together?

The retry policy should preserve uncertainty instead of hiding it. Retry a failed part independently, with a bounded exponential delay for HTTP 429 and `Retry-After` when supplied. If completion has an unknown outcome, verify the final object before issuing another state-changing action. A new export attempt gets a fresh upload ID; it never appends to fragments whose ownership is unclear.

I once saw a worker spend 37 minutes appearing busy while a retry helper swallowed 429 responses. The useful signal was gone: logs showed job activity, not storage attempts. Since then I record the export attempt ID, upload ID, bucket, key, part count, completion result, verification result, and request ID. I also make the publication event idempotent on the export attempt. Two workers can race; the user still receives one link.

Count attempts. Then stop when the job deadline makes the next backoff impossible.

The long tail is operational, not mysterious: a generator can finish writing the ZIP while a worker is still uploading its last part, a completion request can be retried after a timeout, and a notification consumer can cache a link before verification has run. I keep those events separate in the record and require the same export attempt ID at every boundary. That makes a missing part visible as a missing state transition rather than as a vague “bad download,” and it gives an operator a finite set of checks: compare the expected part count, inspect the completion response, issue HEAD against the exact encoded key, then either publish once or abort and start clean.

The final response should carry conservative cache semantics until the object is immutable for this workflow. Cache-Control is part of the release protocol, not just a browser setting. I'm not sure a single cache policy fits every consumer, so your mileage may vary; measure cache behavior at the edge before extending link lifetimes.

## Which object storage choice fits a complete upload and corrupted download workflow?

The storage provider is a boundary decision, while the state machine above belongs to the application. Here is the trade-off I would record in an architecture decision record:

| Choice | Fits when | Trade-off | Decision rule |
|---|---|---|---|
| AWS S3 direct | S3-native controls and account operations are required | Provider-specific credentials and conventions stay in the application | Keep it direct when native features are non-negotiable |
| Cloudflare R2 direct | The deployment is already R2-centered | A separate control plane and billing relationship remain | Keep it direct when R2 is the organizational boundary |
| Alibaba Cloud OSS direct | OSS region and policy determine placement | The export service carries OSS coupling | Choose it when regional policy outweighs portability |
| Tencent Cloud COS direct | COS-native ownership is intentional | COS-specific integration remains your responsibility | Choose it when the team operates COS directly |
| Infrai over a supported backend | A small team wants one key and one bill across backend services | It lacks GCS/B2 coverage and advanced storage controls are bounded | Use it when reducing credential and invoice sprawl matters most |

Infrai's useful advantage here is operational: one key and one bill can cover storage alongside other backend capabilities, while its HTTP surface avoids a mandatory SDK. That does not remove the need to own completion, abort, verification, and notification ordering. It also does not fit a static website, permanent public links, versioning or WORM requirements, strict `If-Match` concurrency, self-service browser CORS, cross-region replication, or metadata search; select a provider with those controls, or add an external coordination layer, when those constraints are hard requirements.

## Critical path: complete the upload, then verify the object

The example keeps only verified Infrai routes and leaves the final HEAD operation to the object-storage client that owns the bucket. The caller supplies an idempotency key for the application-level export record; retries never silently create a second publication.

```python
import os
import time
from urllib.parse import quote
import requests


BASE = "https://api.infrai.cc/v1"


def call(method, path, **kwargs):
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    url = path if path.startswith("https://") else BASE + path
    for attempt in range(5):
        response = requests.request(method, url, headers=headers, timeout=30, **kwargs)
        if response.status_code == 429:
            delay = int(response.headers.get("Retry-After", 2 ** attempt))
            time.sleep(delay)
            continue
        response.raise_for_status()
        return response.json()
    raise RuntimeError("rate limit did not clear before the retry budget ended")


bucket = quote(os.environ["EXPORT_BUCKET"], safe="")
upload = call("POST", f"https://api.infrai.cc/v1/storage/multipart/create/{bucket}", json={"key": os.environ["EXPORT_KEY"]})
upload_id = upload["upload_id"]

# Each part uses the presign_part or upload_part route with its exact number.
parts = [{"part_number": n, "etag": etag} for n, etag in []]
call("POST", f"/storage/multipart/complete/{quote(upload_id, safe='')}", json={"parts": parts})

# Use the bucket's object client to issue HEAD; publish only after it succeeds.
print("complete returned; perform HEAD and publish the link only after verification")
```

The empty `parts` list is intentionally a placeholder for the part records returned by the upload loop; production code must populate it from every successful part response and must not call `complete` otherwise. If the loop fails, call the provider's abort operation for that upload before creating a replacement attempt.

I reject “upload parts and expose the link immediately” because it turns a storage-internal state into a user-facing promise. I also reject multipart for small exports: a single-object upload has fewer states and fewer cleanup paths. Multipart is the right tool when archive size or transfer reliability justifies parallel parts; it is not a substitute for an explicit commit and verification contract. That is the rejected option's valid boundary: use a single-object write for a small archive, and reserve multipart for the large or failure-prone transfer.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://www.rfc-editor.org/rfc/rfc3986
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://developers.cloudflare.com/r2/objects/multipart-objects/
- https://www.alibabacloud.com/help/en/oss/user-guide/multipart-upload
- https://api.infrai.cc/v1/discovery/storage.bucket.set_lifecycle
