# EdTech Browser Uploads: Presigned URLs for Private User PDF/DOCX Documents

Short answer: for an edtech system storing training artifacts, let the browser send bytes directly to private object storage, use multipart transfer for files large enough to make a full retry painful, and let the application database own retention and processing state. A signed download link should be created only after the application has checked the learner, course, and artifact permissions.

The important design choice is where each responsibility lives. The browser handles throughput. The storage layer holds opaque bytes. The application decides who may upload, which course owns the artifact, when a retention period expires, and whether a worker may parse the PDF or DOCX. That boundary keeps a busy Express process from becoming a file relay and keeps a temporary URL from becoming your authorization model.

## Reliability: what survives when a course upload is interrupted?

An uploaded training artifact needs an application record before it needs a download link. Give the record a stable identifier, tenant or school identifier, course identifier, uploader identifier, object key, declared media type, expected size, retention deadline, and a state such as `pending`, `uploaded`, `processing`, `ready`, `expired`, or `rejected`. Generate the object key on the server. Do not use the learner's filename as the key: it is neither unique nor a good place to encode authorization.

Keep it private.

Consider a test fixture that contains a 1.8 GB scanned course pack and a 12 MB DOCX syllabus. The first file should exercise interrupted parts, tab suspension, a repeated completion request, and a cleanup sweep after cancellation; the second should exercise the simpler single-request path. At each boundary, assert the database state and the storage state separately. A browser response with status 200 is not enough if the application still says `pending`, and an object that exists is not enough if it belongs to the wrong server-generated key. Test an unauthorized download as HTTP 403 at the application boundary, an expired record as a denied request, and a duplicate queue delivery as one processing claim. This is a much better test than clicking Upload once on a fast office connection because it covers the moments where a training artifact can become visible to a parser without becoming visible to the learner, or can remain billable after the course has been deleted. The fixture is synthetic; the assertions are the durable part. — That distinction matters when a prompt-cost or retrieval eval later reports a change and you need to know whether the source bytes, the retention job, or the parser caused it.

The browser asks the application for an upload target only after authentication and policy checks. The application can reject an over-quota request, an unapproved media type, or a file whose declared size exceeds the course policy before any large transfer begins. The browser then sends the bytes to the target and calls a completion endpoint owned by the application. Completion is where the server verifies that the expected object exists and advances the record.

That last step is easy to skip. It is also where many document pipelines quietly become unreliable. A browser can finish sending bytes while the completion request is lost; a worker can receive the same notification twice; a user can close the tab after the storage service accepts the final part. Keep the record pending until the server has observed completion, and make the completion operation safe to retry.

The object key is durable. The presigned URL is temporary. Store the first; manufacture the second only for the transfer that needs it.

## Implementation: a transfer ledger for each training artifact

Use one transfer state machine for PDF, DOCX, and any other permitted training artifact. The file extension is a user-interface hint, not a security decision. Check the declared type and size at admission, inspect the content after upload, and keep the object private while those checks run. A parser should read by object identity, never by a user-controlled path.

For a normal-sized file, a single upload is easier to observe and recover. For a large recording transcript, a scanned course pack, or a bundle of lesson documents, multipart upload changes the retry unit: a failed part can be sent again without resending every accepted byte. The client must retain the upload identifier and the accepted part metadata until the server has completed the object. If the learner cancels, abort the multipart session and mark the record abandoned; do not leave the browser responsible for remembering cleanup forever.

Here is the small part of the state machine that should be testable without a cloud account. The storage adapter is deliberately an interface: its job is to create, complete, or abort a transfer, while the database transaction protects the training-artifact record.

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


class ArtifactState(str, Enum):
    PENDING = "pending"
    UPLOADED = "uploaded"
    PROCESSING = "processing"
    READY = "ready"
    EXPIRED = "expired"
    REJECTED = "rejected"


@dataclass
class Artifact:
    artifact_id: str
    course_id: str
    object_key: str
    retention_until: datetime
    state: ArtifactState = ArtifactState.PENDING


def complete_artifact(artifact: Artifact, observed_key: str) -> None:
    """Promote an object only when it matches the server-owned record."""
    if artifact.state is not ArtifactState.PENDING:
        return  # A retried completion must not enqueue the artifact twice.
    if observed_key != artifact.object_key:
        raise ValueError("completed object does not match artifact record")
    artifact.state = ArtifactState.UPLOADED


def claim_for_processing(artifact: Artifact) -> bool:
    """Return true for one worker claim and false for duplicate delivery."""
    if artifact.state is not ArtifactState.UPLOADED:
        return False
    artifact.state = ArtifactState.PROCESSING
    return True
```

In production, `complete_artifact` belongs behind a transaction or compare-and-set update, and the worker claim needs the same protection. The point of this example is the invariant, not a pretend storage SDK. I use an eval harness for extraction and retrieval, so I also keep the original object key, content digest when available, parser version, and prompt-cost metadata with the processing result. That makes a change to chunking or prompts measurable against the same source file instead of a new upload.

## Measure large-file recovery before choosing multipart

Throughput is more than bandwidth. It is the amount of useful work preserved when a connection drops, a tab is suspended, or a mobile network changes. A single request has a simple control path but a costly retry. Multipart has more state and better failure locality. Choose the boundary with a measured transfer test that reflects your learners' networks, rather than copying a threshold from another product.

The browser should limit concurrent parts, retain progress by part number, and retry only a failed part. It should not retry blindly after an ambiguous response: first ask the application whether the upload is still pending, then continue with the server's record of accepted parts. The completion call must include the exact ordered part metadata required by the storage service. An application-level idempotency key should cover creation and completion so a refresh cannot produce two artifact records.

The server also needs a sweeper. It should find pending records past their admission deadline, abort unfinished multipart transfers, and mark the records abandoned or rejected according to policy. Storage lifecycle rules can help with old objects, but they are not a substitute for a database state transition, and a rule for completed objects does not necessarily account for unfinished multipart state. Log artifact ID, course ID, transfer ID, part count, bytes, and final state; never log signed URLs or bearer credentials.

One operational detail matters for RAG: do not enqueue parsing merely because the browser displayed 100 percent. Enqueue after server-side completion, and make the consumer idempotent by artifact ID and object key. A duplicated queue message should repeat a safe read, not create a second set of chunks in the index.

## Retention policy is a data-governance record

A download request should authenticate the caller, authorize access to the course and artifact, check that the artifact is ready and within its retention window, and then issue a short-lived signed GET capability. The link is a delivery mechanism. It is not proof that the requester is allowed to see the document.

Retention has two clocks. The application clock controls whether a learner may request the artifact. The storage clock controls when bytes are removed. Delete the object and update the record in an order that a retry can reconcile, then make the download path reject an expired record even if an old link remains in a browser history. Keep the retention deadline in UTC and test the boundary around midnight and daylight-saving transitions.

Private storage is a poor fit for a permanent public course catalog, anonymous asset hosting, or a workflow that requires immutable, provider-managed records without another control plane. In those cases, choose a storage and governance design with the required public-delivery or retention primitives, and keep learner documents in a separate private namespace. A signed URL also does not stop a permitted learner from copying the file; watermarking, viewer controls, and downstream policy are separate concerns.

For regulated education programs, treat the authorization record, audit trail, deletion policy, and incident process as part of the design. FedRAMP is a federal cloud security authorization program, not a blanket certificate for every storage configuration; its program material is useful when mapping controls, but it does not remove the need to document your own deployment boundary.

## How should browser direct upload serve private user documents with presigned URLs?

The request sequence is short enough to draw on a whiteboard: authenticate the user, create a server-owned artifact record, issue a temporary upload capability, transfer bytes, confirm the object, enqueue processing, authorize a later read, and issue a temporary download capability. The application owns the transitions. Storage should not have to know that a file belongs to course 314 or that a learner has left the class.

For browser direct upload, the exact allowed origin, methods, headers, and credential behavior must be part of the deployment test. If cross-origin policy cannot be configured to the required standard, send the bytes through an application gateway and accept the bandwidth cost deliberately. This is an environment constraint, not a reason to make the storage namespace public.

The same policy applies to a DOCX as to a PDF. A file type can determine which parser runs later, but it must not bypass authorization, retention, malware inspection, or audit logging.

## Compare the boundary: proxy, direct, or multipart

Start with a private bucket or equivalent namespace, server-generated keys, an application retention record, and signed downloads. Add multipart transfer when the restart cost of a full upload is material. Keep an Express proxy as an intentional fallback for environments where browser cross-origin policy cannot be configured or where the application must inspect bytes before storage. The proxy costs application bandwidth and operational capacity, so it should be a policy choice rather than an accidental default.

The catch is that direct upload is not suitable when your security review requires all bytes to pass through a controlled inspection tier, and multipart is not worth its state machine for small files. Stick with a native storage integration when its identity, residency, retention, or replication controls are already the thing your review depends on. Use a gateway layer when the main value is a common HTTP contract across several capabilities, but verify its limits before making it the system of record.

| Transfer shape | Use it when | Main trade-off |
| --- | --- | --- |
| Single direct upload | The file is small enough that a retry is acceptable | The retry unit is the whole file |
| Multipart direct upload | Large-file throughput and interrupted-transfer recovery matter | The client and server must track parts, completion, and aborts |
| Application proxy | Inspection or browser policy requires a controlled byte path | The application tier pays for ingress, egress, and connection capacity |

This table is a starting decision rule, not a benchmark. Measure interrupted uploads, repeated completion calls, and download authorization with representative training artifacts before setting a production threshold.

I’m not sure which upload threshold will hold for every school network; that is exactly why the transfer benchmark belongs in the repository beside the eval harness. Test interrupted parts, repeated completion calls, expired links, unauthorized course access, parser retries, and deletion races. Then make the decision from observed failure cost, not from the upload component's progress bar.

## References

- https://developers.cloudflare.com/r2/
- https://www.fedramp.gov/
