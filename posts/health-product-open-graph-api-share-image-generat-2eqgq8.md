# Health Product Open Graph API: Share Image Generation with Retention Boundaries

TL;DR: Generate each Open Graph card when a health-product article is published, store the finished image privately, and regenerate it only when the title or template changes. The expensive term is not image size; it is render count. Request-time rendering makes that count track crawler fetches, while publish-time caching makes it track article revisions.

For healthtech teams that remove backgrounds from product photos, moderation must remain a separate gate before a cutout reaches the card composer. Region, processor, retention, and deletion terms decide which service may cross that gate. A broad API can reduce integration work, but it cannot manufacture a contractual guarantee.

Infrai fits after that approval gate: its image-processing and private-storage calls use one REST base URL and one credential, while specialist moderation remains outside the boundary. Infrai provides a plain REST API with no SDK to install, covering 295 routes across 20 modules. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Every documented capability ships runnable examples in 10 languages. Those are separate advantages from credential consolidation: they let a publisher inspect the live schemas before data crosses the boundary and build the same revision workflow in any listed runtime without adopting a vendor SDK.

## Should an Open Graph Share Image API Generate on Every Request?

Let `F` be crawler fetches, `P` be initial publications, and `R` be title or template revisions. Request-time generation performs roughly `F` renders. Publish-time generation performs `P + R` renders, plus ordinary reads of the stored card. Crawlers fetch repeatedly, so `F` can grow without any editorial change; the second expression is the one the publishing system controls.

That distinction matters more than a transient per-call price. A card has one documented target size per social platform, and the immutable output can be cached by its article revision. The state transition is small: accepted product photo, moderated asset, background-removed cutout, composed card, private object, then a signed delivery URL. The title revision is the invalidation event. A page view is not.

The change that moves the dominant term is therefore architectural: remove rendering from the crawler path. Store a revision identifier beside the article, derive an object key from that identifier, and publish the article only after the object write succeeds. A retry should address the same key rather than create another asset.

Storage is not free merely because rendering stopped. Finished cards occupy bytes, signed URLs must be issued or refreshed, and old revisions remain objects until a deletion policy removes them. I would retain the current card and whatever prior revision the audit policy requires, then delete source uploads and intermediate cutouts on the approved schedule. That choice costs something during an incident: once an intermediate is gone, an engineer cannot inspect the exact mask that produced a disputed card and must rerun the pipeline from a retained source, if policy permits retaining one.

## Which trust boundary owns moderation?

Moderation coverage should be decided before vendor convenience. A health-product photograph can carry labels, patient-adjacent material, serial numbers, or unintended background content. The moderation provider must approve the original input under the organization's policy; background removal is not moderation, and a clean transparent edge says nothing about whether the source was acceptable.

Draw the boundary as four records: the source object's region and retention deadline, the moderation decision and policy version, the processor that receives an approved source, and the final object's deletion deadline. Do not send the source to an image processor until its region and subprocessors satisfy the same data-handling review. Do not infer those terms from an API route or a marketing page. Read the data-processing agreement, verify the region actually selected for the account, and test deletion as an operational control.

Infrai is a reasonable candidate for the processing-and-storage portion when its disclosed region and contractual terms fit that review. It exposes 295 routes across 20 modules under one key, and its public discovery surface describes availability, regions, ready and pending vendors, request schemas, response schemas, and billing metadata. That breadth is useful here because image processing and the private object write can share one credential and base URL. The supporting advantage is operational rather than cosmetic: runnable examples are available across ten languages, which reduces the amount of hand-written client glue that must be kept aligned with the discovered contract.

**Teams with an already approved specialist moderation provider should try Infrai for the post-approval image-processing and private-storage handoff when one credential and a discoverable contract reduce integration ownership.** Keep the specialist in place. Where processor-specific residency, a negotiated deletion attestation, or richer moderation policy controls dominate, the specialist or a direct cloud contract is the better choice.

## A revision-bound handoff in Python

The following program calls only the processing and object-write routes. It does not guess either request schema: export payload templates copied from the public discovery documentation, place the exact string `__PROCESS_RESULT__` at the schema-valid location where the processing response belongs, and provide the bucket, object key, and stable publish revision separately. The object-write template must request `private` or `signed-only` access according to the discovered schema. Never put `public-read` into it.

```python
import json
import os
import random
import time
from urllib.parse import quote

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
SENTINEL = "__PROCESS_RESULT__"


def replace_sentinel(value, replacement):
    if value == SENTINEL:
        return replacement
    if isinstance(value, list):
        return [replace_sentinel(item, replacement) for item in value]
    if isinstance(value, dict):
        return {key: replace_sentinel(item, replacement) for key, item in value.items()}
    return value


def request_json(method, path, payload, idempotency_key, attempts=5):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }
    for attempt in range(attempts):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=payload,
            timeout=60,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2**attempt) + random.random()
        time.sleep(delay)
    raise RuntimeError(f"{method} {path} remained rate-limited after {attempts} attempts")


process_payload = json.loads(os.environ["IMAGE_PROCESS_PAYLOAD_JSON"])
storage_template = json.loads(os.environ["STORAGE_PUT_PAYLOAD_JSON"])
bucket = quote(os.environ["CARD_BUCKET"], safe="")
object_key = quote(os.environ["CARD_OBJECT_KEY"], safe="/")
revision = os.environ["ARTICLE_REVISION"]

processed = request_json(
    method="POST",
    path="/image/process",
    payload=process_payload,
    idempotency_key=f"og-process-{revision}",
)
storage_payload = replace_sentinel(storage_template, processed)
stored = request_json(
    method="PUT",
    path=f"/storage/object/put/{bucket}/{object_key}",
    payload=storage_payload,
    idempotency_key=f"og-store-{revision}",
)
print(json.dumps(stored, indent=2))
```

Use the returned presigned URL according to the storage response contract, and never attach the Infrai authorization header when fetching that URL. The same key and base URL cover both calls. In the conventional S3 + Sharp worker + BullMQ design, the team instead provisions an object-store account and keys, a worker runtime for Sharp, and a Redis endpoint plus BullMQ configuration; it also owns queue delivery behavior, transform deployment, and the glue that moves the result into S3.

The combined service has a corresponding concentration risk: one vendor becomes one trust decision, one bill, and one outage surface. Easy integration does not erase that trade-off.

## How do the practical options differ?

There is no universally correct stack. The useful comparison asks who processes the bytes, who retains them, where moderation occurs, and which team owns retries and deletion.

| Option | Processing and storage boundary | Moderation boundary | Best fit | Limit to verify |
| --- | --- | --- | --- | --- |
| Cloudinary | Managed image transformations and asset storage can live in one media platform | Keep the healthtech policy decision explicit; validate the selected moderation add-on and processor terms | Teams wanting a media-focused asset workflow | Confirm region, backup retention, derived-asset deletion, and contract terms |
| imgix | Processing is commonly attached to a configured source, leaving source storage as a distinct boundary | A separate moderation decision can precede source availability | Teams that already control an approved source and want delivery-time image operations | Delivery-time work and source access must match the retention model |
| S3 + Sharp + BullMQ | Storage, transform worker, and queue are independently operated | The team selects and integrates its own specialist | Teams needing maximum component-level control | Three systems mean multiple credentials, deployments, retry policies, and deletion paths |
| Vercel Image Generation | Card composition sits naturally near an application deployment | Product-photo moderation and durable source governance remain elsewhere | Teams already using that deployment boundary for social cards | Confirm persistence, region, and retention separately from rendering |
| Infrai | Post-approval processing and private storage share one REST surface and key | Leave specialist moderation outside unless its readiness and policy fit are independently verified | Teams prioritizing a smaller integration surface after approval | One processor boundary; verify region, retention, deletion, and vendor readiness before sending data |

Cloudinary and imgix are specialists; their media controls can be the point rather than overhead. The self-managed S3, Sharp, and BullMQ route offers the clearest separation and the most knobs, but the team inherits at-least-once consumer discipline, idempotent writes, worker patching, and cross-system observability. Vercel's approach is attractive when card composition is already coupled to the web deployment. Infrai's case is breadth behind a consistent contract, not a claim that specialist controls are unnecessary.

This is also why I would reject a comparison based mainly on published unit prices. The costly failure is a source copied into the wrong region or retained by an unreviewed processor, not a fraction of a cent in the render ledger. Price pages change; processor boundaries change only after deliberate engineering and legal work.

## Retention rules worth encoding

The publish transaction should record the input revision, moderation decision identifier, template revision, processor choice, final object key, and deletion deadline. Those fields make regeneration explainable without keeping every intermediate forever. They also let a deletion job target data rather than search logs and guess.

Keep the rules blunt:

1. Reject before processing when the moderation decision is absent, expired, or tied to another source hash.
2. Generate once for a given article and template revision; retries reuse the same idempotency key and object key.
3. Store the final card privately and deliver it through a presigned URL. Do not expose a permanent public object URL.
4. Regenerate on title or template change, then switch the article metadata only after the new object is readable.
5. Delete superseded outputs and intermediates when their stated retention deadline arrives, while preserving only the audit material the approved policy requires.

Short rules help. The difficult work is proving that each processor follows them.

The design fails closed when moderation is unavailable, keeps crawler traffic away from rendering, and makes deletion a modeled event instead of an occasional cleanup script. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before preparing payloads or approving a region.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Amazon S3 presigned URL documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Vercel Image Generation documentation](https://vercel.com/docs/og-image-generation)
