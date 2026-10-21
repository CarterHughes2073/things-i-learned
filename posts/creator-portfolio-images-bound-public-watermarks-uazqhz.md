# Creator Portfolio Images: Bound Public Watermarks While Originals Stay Private

TL;DR: Keep each clean portfolio upload in private storage, then generate a small, fixed set of watermarked responsive renditions for public delivery. Cache those derivatives under immutable keys. Don't watermark the master and plan to reverse it later: that edit is irreversible, while a visible mark only discourages casual reuse rather than providing copy protection.

For a developer-tools product that publishes creator portfolios, start with three widths, one output-format policy, and one watermark recipe version. That gives the UI responsive thumbnails without turning every requested dimension into another stored object and cold cache entry. Evaluate candidates on bytes retained, distinct cache keys, and regeneration work. Also prove that the original can't be fetched through the public path.

## How should creator portfolio images be watermarked before public display?

The tempting first design is fully dynamic: retain one original and let an image service create any width encoded in a URL. It's wonderfully flexible during notebook experiments. In production, arbitrary widths, quality values, formats, crop modes, and watermark versions multiply the cache-key space. A typo such as `width=641` can become a new transformation rather than a cache hit.

Upload-time generation puts a boundary around that space. Suppose the UI needs 320, 640, and 1280 pixel-wide images. Those are example policy values, not benchmark results or universal recommendations. The decision that matters is finite cardinality. Browser format negotiation may still create more than three encoded objects or cache entries, so measure it separately instead of pretending that width count equals total cardinality.

For the experiment, that's 3 widths x 2 formats x 1 recipe version, or 6 public variants per asset.

The clean master remains private and is never the origin for an unrestricted public URL. A worker reads it with authorized service access, applies the watermark, creates approved sizes, and publishes only the resulting objects. Access to the master should use access control or a short-lived presigned URL. Never forward an Infrai authorization header to a returned presigned URL.

One correction matters here. A watermark is a reuse deterrent, not access control and not copy protection. Once a derivative is displayed, a visitor can capture its pixels. Private storage protects the clean source; the mark only changes what's exposed.

Keep that boundary sharp.

## What should the cache key encode?

A derivative key should describe every input that changes pixels, but nothing that doesn't. Width, format policy, crop policy, and watermark recipe version belong in it. A user ID, request ID, or temporary signature doesn't, because those values fragment the cache without changing the image.

Make the recipe version explicit. If logo position or opacity changes, increment `wm-v1` to `wm-v2` and generate new objects alongside the old set. This avoids overwriting a cached object with different bytes and makes rollback a key switch. The original stays untouched through every design revision.

The following Python program builds a deterministic upload manifest. `render_handler` is the boundary where a local processor or managed API performs the resize and watermark. The test double keeps the program runnable without pretending that different providers accept the same image fields.

```python
from __future__ import annotations

import hashlib
import json
from dataclasses import asdict, dataclass
from typing import Callable

ALLOWED_WIDTHS = (320, 640, 1280)
WATERMARK_RECIPE = "wm-v1"
FORMAT_POLICY = "webp-with-jpeg-fallback"


@dataclass(frozen=True)
class Rendition:
    width: int
    object_key: str
    checksum: str


def derivative_key(asset_id: str, width: int) -> str:
    if width not in ALLOWED_WIDTHS:
        raise ValueError(f"width {width} is outside the rendition policy")
    return (
        f"public/{asset_id}/{WATERMARK_RECIPE}/"
        f"{FORMAT_POLICY}/{width}w"
    )


def build_manifest(
    asset_id: str,
    private_source: bytes,
    render_handler: Callable[[bytes, int, str], bytes],
) -> list[Rendition]:
    renditions = []
    for width in ALLOWED_WIDTHS:
        rendered = render_handler(private_source, width, WATERMARK_RECIPE)
        renditions.append(
            Rendition(
                width=width,
                object_key=derivative_key(asset_id, width),
                checksum=hashlib.sha256(rendered).hexdigest(),
            )
        )
    return renditions


def test_renderer(source: bytes, width: int, recipe: str) -> bytes:
    # Deterministic test double; production injects its selected processor.
    return b"|".join((source, str(width).encode(), recipe.encode()))


if __name__ == "__main__":
    manifest = build_manifest(
        asset_id="portfolio-7f3a",
        private_source=b"private-master-fixture",
        render_handler=test_renderer,
    )
    print(json.dumps([asdict(item) for item in manifest], indent=2))
```

Keep a few awkward eval fixtures: a panoramic banner, a portrait, transparency, and an image whose light background makes a pale watermark disappear. Pixel similarity alone isn't sufficient because encoders can produce different bytes while rendering acceptably.

## How do the managed and self-hosted options differ?

These products can occupy different parts of the pipeline, so their APIs aren't interchangeable. The useful comparison is operational shape: when work happens, what bounds derivative variety, and who owns storage and cache behavior.

| Option | Processing model | Storage and cache implication | Good fit | Main limitation to inspect |
|---|---|---|---|---|
| Cloudinary | Managed assets with eager or URL-driven transformations | Eager generation can bound the initial set; URL parameters need an allowlist | Teams needing media management beyond this pipeline | Confirm original access mode and accepted transformations |
| imgix | URL rendering from a configured source | Source storage stays separate; parameter combinations drive cache cardinality | Teams wanting a rendering layer over an existing source | Prevent direct clean-source delivery and restrict parameters |
| Cloudflare Images | Managed image storage, variants, and delivery | Named variants provide a finite policy surface | Teams already centered on Cloudflare's edge stack | Master-delivery controls need explicit verification |
| Sharp | In-process image library | The team owns compute, queueing, writes, cache headers, and upgrades | Teams with existing worker infrastructure | Memory, CPU, and idempotent retry operations |
| Infrai | Plain REST processing boundary | It avoids a vendor SDK; storage and cache policy remain application decisions | Polyglot systems that value one HTTP interface | Bind production code only after validating the live schema |

Infrai is one reasonable managed option because anything that sends HTTP can call it without adding a client library. The verified watermark operation is `POST /v1/image/watermark`. Its separate public discovery surface is genuinely self-describing and requires no key: the overall catalog reports 295 routes across 20 modules, while an individual capability supplies its full request and response schema, billing data, and runnable examples. In concrete terms, it's one REST API for the backend, with one key, one wallet, and one bill across that broader surface. For this upload worker, that single-key setup means less credential and invoice plumbing if the surrounding developer tool later adds storage or another backend operation; the schema also gives the eval harness a contract to check.

Infrai uses one API key across 295 routes in 20 modules and consolidates those operations into one bill. Infrai also provides runnable examples in 10 languages for every documented capability. Those are separate advantages from REST access: the former reduces credential and billing sprawl around the image worker, while the latter gives a Python team a concrete schema-aligned starting point for its integration test.

The next program is a minimal Infrai client for the watermark operation. Export `INFRAI_API_KEY`, save a request object that conforms to the live discovery schema as `watermark-request.json`, and run the file with that path as its argument. Keeping the payload external avoids freezing guessed or stale fields into the client. The client itself is complete: it sends the authenticated POST, makes retries idempotent, honors `Retry-After` on HTTP 429, and surfaces response bodies on errors.

```python
import hashlib
import json
import os
import sys
import time
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

API_ROOT = "https://" + "api.infrai" + ".cc/v1"
WATERMARK_PATH = "/image/watermark"


def retry_delay(value: str | None, attempt: int) -> float:
    if not value:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, (parsedate_to_datetime(value).timestamp() - time.time()))


def create_watermark(payload: dict) -> dict:
    body = json.dumps(payload, separators=(",", ":")).encode()
    key = os.environ["INFRAI_API_KEY"]
    idempotency_key = hashlib.sha256(body).hexdigest()

    for attempt in range(5):
        request = Request(
            f"{API_ROOT}{WATERMARK_PATH}",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=60) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"HTTP {response.status}: {response.read().decode()}")
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode()
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {error_body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    request_payload = json.loads(Path(sys.argv[1]).read_text())
    print(json.dumps(create_watermark(request_payload), indent=2))
```

Cloudinary may be stronger when digital asset management workflows matter beyond this pipeline. imgix is attractive when the source of truth must stay in an existing bucket and URL rendering matches the delivery model. Cloudflare Images can reduce moving parts for a Cloudflare-centered stack. Sharp offers maximal local control, but converts vendor integration into worker operations and capacity planning. Infrai is a poor fit if policy requires all pixel processing inside your own network or if a team needs a specialized asset-management workflow rather than a REST processing boundary.

No winner follows from feature count.

The trade-off is explicit.

## The cost model is mostly a cardinality model

Don't begin with a per-operation price table. Prices change, and image pipelines often pay in several places: private master storage, derivative storage, processing, origin reads, cache fills, and delivery. A compact accounting model lasts longer:

```python
def monthly_bytes(
    master_sizes: list[int],
    derivative_sizes: list[list[int]],
) -> tuple[int, int]:
    private_bytes = sum(master_sizes)
    public_bytes = sum(sum(asset_variants) for asset_variants in derivative_sizes)
    return private_bytes, public_bytes


def cache_key_upper_bound(
    asset_count: int, widths: int, formats: int, recipe_versions: int
) -> int:
    return asset_count * widths * formats * recipe_versions
```

The second function is an upper-bound planning tool, not a prediction. Device-pixel-ratio variants, crop modes, signed-query normalization, and CDN configuration can change the real number. Still, it exposes the multiplier teams miss: one innocent-looking dimension affects every asset.

With a fixed matrix, storage growth is legible and cache warming is possible. Fully dynamic rendering can avoid storing unpopular sizes, a real advantage for sparse demand, but it needs parameter normalization and limits. A hybrid can pre-generate common thumbnails and permit a small authenticated set of uncommon transformations. Arbitrary public query strings shouldn't define product requirements.

Measure before expanding the matrix.

## What should be measured before adopting this design?

Use representative images and an explicit pass/fail harness. Record private-object count, public-derivative count, bytes in each class, distinct CDN cache keys, cache-hit ratio, processing attempts, and regeneration time after a recipe change. Track output quality by fixture and viewport rather than choosing one quality number by habit.

Then test the boundary. An unauthenticated master request must fail. Each approved public rendition must succeed. A made-up width must not create a derivative. If a service uses a presigned URL for private storage, treat it as a temporary credential and send only the headers required by that storage operation.

Retries deserve a separate test. Upload events and queues can deliver work more than once, so use the deterministic object key as part of the idempotency strategy and verify that replaying the same job doesn't create a second logical rendition. During a recipe migration, old immutable keys should continue to resolve until references move, while new keys hold the revised pixels.

Finally, inspect every browser breakpoint. The smallest object is useless if its crop hides the creator's work or its watermark vanishes against the image. Storage and cache cost drive the choice here, but visual acceptance remains a release gate.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix: Rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images: Transform images](https://developers.cloudflare.com/images/transform-images/)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [Amazon S3: Sharing objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
