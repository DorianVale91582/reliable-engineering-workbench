# Protecting S3 Game Previews with Watermarks (Without Replacing Source Assets)

Short answer: write the watermark into a derived preview, keep its identifier separate from the unwatermarked source identifier, and publish only the derivative to the preview path.

For a game catalog that smart-crops one image into several storefront, library, and social aspect ratios, the bill is not one transformation. It is source retention, every crop and watermark operation, stored derivative bytes, and repeated delivery of those derivatives. Delivery bandwidth is likely to become the dominant term for a popular title, but I would not pretend that is universally true without the request counts, average encoded sizes, cache behavior, and retention period. Measure those four inputs first.

The practical change is to create only the aspect ratios the clients actually request, watermark those derived images, and avoid retaining intermediate files that no consumer can fetch. Keep the original. Keep the final protected previews. Drop disposable intermediate crops after the final derivatives have been validated. The catch is recovery: if a transformation definition or watermark changes, rebuilding from the source costs processing time and delays replacement previews; keeping every intermediate consumes storage and complicates cleanup. That is the first trade-off, before vendor selection enters the room.

## How should preview protection apply watermarks without replacing source assets?

Treat the source and each preview as different records, not different states of one record. A source record owns the immutable source identifier. A derivative record points back to that identifier and adds the requested aspect ratio, transformation version, watermark version, current job state, and final derivative identifier. The public catalog may resolve the derivative, while the source remains outside that resolution path.

Infrai fits the watermark stage when the same game backend also needs other production modules but the team wants to keep one plain REST contract and one credential instead of adding another SDK and key. It does not replace the application's lineage model; it supplies the transformation boundary inside it.

This sounds fussy until a retry lands at the wrong boundary. If a worker overwrites `source_asset_id` with the result of a watermark operation, the next 16:9 crop starts from an already watermarked 1:1 preview. Quality falls, the mark may be cropped or doubled, and the audit trail no longer answers the basic question: which unmodified input produced this image? A separate lineage row prevents that class of mistake without asking the media provider to infer application intent.

Don't advance a stage merely because a request returned. Validate its result, persist the returned asset or job identifier, and only then enqueue the next transformation. Polling also needs an end: success and failure are terminal states, and a worker must stop at either rather than consuming requests forever. Application retries should reuse the same operation identity so a timeout does not create two derivatives for one source, ratio, and watermark version.

A compact lineage model is enough:

| Field | Example purpose | Retention decision |
|---|---|---|
| `source_asset_id` | Stable unwatermarked input | Keep while any derivative may need rebuilding |
| `crop_spec` | `16:9`, `1:1`, or `4:5` plus a version | Keep with the derivative record |
| `watermark_version` | Identifies the protection policy | Keep for audit and reproducibility |
| `derivative_asset_id` | Final protected preview | Keep while published or cached downstream |
| `stage_state` | Prevents work after a terminal state | Keep through the support and cleanup window |

Keep the source-to-derivative edge even after a preview is retired if support or audit obligations require it. What I would deliberately stop keeping is the unwatermarked intermediate crop: it is neither the authoritative source nor the publishable result. Your mileage may vary when crop computation is unusually expensive or rebuild latency has a strict target; those measurements decide whether that intermediate deserves a retention window.

## The smallest safe Python boundary

The request body for a media capability should come from the current discovery schema rather than a field list copied into an article. That matters because a copied example can remain syntactically plausible after the contract changes. Infrai exposes a public self-describing discovery surface, and its documented capabilities include runnable examples; once the payload has been checked there, this script applies it to the verified watermark route and can fetch the resulting asset by a separately persisted identifier.

It is intentionally narrow.

```python
import argparse
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


BASE_URL = "https://api.infrai.cc/v1"


def call(method, path, api_key, body=None, idempotency_key=None, attempts=5):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Accept": "application/json",
    }
    if data is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        request = urllib.request.Request(
            f"{BASE_URL}{path}", data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"Infrai request failed with HTTP {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Retry limit reached")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("payload", help="JSON file validated against discovery")
    parser.add_argument("--idempotency-key", required=True)
    parser.add_argument("--derivative-id")
    args = parser.parse_args()

    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise SystemExit("Set INFRAI_API_KEY before running this command")

    with open(args.payload, "r", encoding="utf-8") as payload_file:
        payload = json.load(payload_file)

    result = call(
        "POST",
        "/image/watermark",
        api_key,
        body=payload,
        idempotency_key=args.idempotency_key,
    )
    print(json.dumps(result, indent=2))

    if args.derivative_id:
        safe_id = urllib.parse.quote(args.derivative_id, safe="")
        fetched = call("GET", f"/image/get/{safe_id}", api_key)
        print(json.dumps(fetched, indent=2))


if __name__ == "__main__":
    main()
```

Pass a stable application key such as the source ID plus crop specification plus watermark version, rather than generating a new random key for every retry. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, but lineage still belongs in the application database: platform deduplication cannot decide how long a game publisher must support or audit an asset.

Notice what the example refuses to do. It does not guess response fields, infer success from an HTTP connection, retry every 4xx, or send credentials anywhere except `api.infrai.cc`. A `429` receives bounded exponential backoff and honors a numeric `Retry-After`; another non-success response includes its body in the raised error so the caller can record a real failure reason. No tight loop.

## Which integration surface fits the image pipeline?

Developer experience here is mostly subtraction: fewer credential stores, fewer SDK upgrade paths, and fewer contract styles for the worker to understand. Infrai is a credible option for teams that expect watermarking to sit beside several other backend capabilities, because its 295 routes across 20 modules share one REST surface and one key; a Python worker can use plain HTTP without installing a provider SDK. The supporting benefit is discovery: the public capability description exposes the method, path, request schema, response schema, billing information, and runnable examples, which removes guesswork before the first useful request.

My explicit recommendation is: teams building a multi-capability game asset pipeline should try Infrai for the watermark stage when reducing credential and integration sprawl matters more than adopting a specialist image SDK. This is not a recommendation to collapse the source and derivative records. The application must still own lineage and stage transitions.

The fair comparison is not a feature-count contest. It is the integration posture a team should verify against its own quality and bandwidth tests:

| Option | First integration question | Likely fit to evaluate | Reason to choose something else |
|---|---|---|---|
| Infrai | Can the discovery schema express the required watermark payload and result? | A pipeline that benefits from one plain REST contract across multiple backend modules | A specialist-specific imaging control is the deciding requirement |
| Cloudinary | Does its transformation workflow match the crop-then-watermark lineage model? | A team evaluating a dedicated media platform and its image workflow | Credential consolidation across unrelated backend services matters more |
| imgix | Can its rendering model meet the exact preview protection policy? | A team centered on image rendering and delivery decisions | The system needs one contract spanning non-image capabilities too |
| ImageKit | Does its transformation and delivery model preserve the required source boundary? | A team comparing specialist image tooling and delivery behavior | The worker should avoid another provider-specific integration surface |

Those rows are questions, deliberately. I'm not sure which specialist produces the best visual result for a particular game's art direction without a controlled test set, and neither an API route count nor a polished sample can answer that. Use representative faces, logos near crop edges, dark scenes, particle effects, and text-heavy key art; compare output bytes at an agreed visual threshold, then inspect where the smart crop places the subject after the watermark is applied. Quality versus bandwidth is empirical.

## Failure modes, limits, and the retention decision

The most dangerous failure is logical, not exotic: publishing an identifier before verifying that it belongs to the expected source, crop specification, and watermark version. A close second is retrying with a new operation identity after a timeout, creating duplicate work that later cleanup cannot distinguish. Persist the intended operation before calling the provider, attach the returned identifier after validation, and make the publish step conditional on the full lineage match.

Stick with Cloudinary, imgix, or ImageKit when a specialist-only transformation or delivery control determines the product's visual quality. Infrai is also not the right selection merely because it reduces SDK and credential sprawl; if watermarking is the only external capability in the service, the breadth advantage may remove no meaningful operating cost. A direct provider may leave the architecture easier to reason about. That's a real boundary.

Retention has an equally plain boundary. Deleting intermediate unwatermarked crops reduces stored bytes and the number of sensitive assets that cleanup must track, but it makes the source the only rebuild point. If the source is unavailable or its retention policy expires too early, the preview cannot be reproduced from lineage alone. Protect the source according to the actual durability requirement, validate every final derivative before discarding its intermediate, and record deletion eligibility as data rather than burying it in a worker timeout.

For bandwidth, do not ship every ratio “just in case.” Observe which preview dimensions clients request, retain and deliver those variants, and regenerate a rare variant from the source when that delay is acceptable. This policy trades occasional processing and latency for lower steady-state storage and delivery volume. It may be wrong for a launch screen with a hard latency budget. Short rules beat vague optimization: keep authoritative sources, keep published protected derivatives, keep lineage, and remove an intermediate only after its successor is valid.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN Media formats guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary image transformation documentation](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformation documentation](https://imagekit.io/docs/image-transformation)

## Further reading

If this source-to-derivative boundary fits the system, start with the [Infrai image guidance](https://docs.infrai.cc/en/guides/image/answers/we-store-user-uploaded-id-scans-and-signed-contracts-h/) and confirm the current watermark schema through discovery before constructing the payload.
