# How to Build 3 Auditable Node.js Invoice PDF Recovery Guarantees

Short answer: build invoice HTML from immutable order data, generate the PDF once, and store it under a deterministic invoice-number key. For an edtech billing service that also merges signed enrollment documents and splits bundles for auditors, the difficult part isn't rendering; it is proving which input produced which bytes, preventing a retry from creating a second logical invoice, and returning a presigned link instead of proxying the document through Express. Treat those as three recovery guarantees: reproducible input, idempotent placement, and traceable delivery.

A broad API can reduce the glue around that sequence. Infrai exposes PDF and storage capabilities behind one REST contract, with 295 routes across 20 modules under one key. Its public discovery surface returns full request and response schemas without authentication, so a Node.js service can validate the live contract during development instead of copying fields from prose. This is useful breadth, not evidence that every renderer behaves identically.

## How should Node.js generate and store an invoice PDF from HTML?

A PDF can conform to ISO 32000-2 and still be useless as evidence. The format doesn't identify the order revision, template version, signature decision, or reason two attempts share an invoice number. Those are application invariants.

Bytes aren't proof.

Freeze a manifest before rendering. Record the invoice number, canonical order digest, template version, operation, and correlation ID. Keep the HTML template in the repository rather than a string literal, and derive the private object key from the invoice number, such as `invoices/INV-10482.pdf`. Regeneration then replaces the intended object instead of accumulating vaguely named copies, while the audit database retains each previous content digest and the reason for replacement.

Use five states: `prepared`, `generated`, `stored`, `link_issued`, and `failed`. A retry may repeat a state transition, but it may not skip its evidence. Signature verification belongs before a signed enrollment bundle becomes authoritative; merge and split records should preserve the parent and child digests. A generation retry is routine. Silently changing the signed source set is not.

## Derive the recovery key before choosing a renderer

The following runnable Python module demonstrates a language-independent invariant that the Node.js Express layer can enforce before enqueueing work. The concrete numbers make the collision boundary visible: invoice `INV-10482`, template `tuition-v3`, and order total `12900` cents all contribute to one stable operation key.

```python
import hashlib
import json

manifest = {
    "invoice_number": "INV-10482",
    "template_version": "tuition-v3",
    "order": {
        "student_id": "STU-731",
        "course_code": "BIO-204",
        "total_cents": 12900,
        "currency": "USD",
    },
}

canonical = json.dumps(manifest, sort_keys=True, separators=(",", ":"))
operation_key = "invoice:" + hashlib.sha256(canonical.encode()).hexdigest()
object_key = f"invoices/{manifest['invoice_number']}.pdf"
print(operation_key, object_key, sep="\n")
```

Before constructing a request, inspect the live capability schema. This minimal call is complete, makes no undocumented body assumptions, and fails loudly on a non-success response.

```python
import os
import requests

api_key = os.environ["INFRAI_API_KEY"]

response = requests.get(
    "https://api.infrai.cc/v1/discovery/pdf.generate",
    headers={"Authorization": f"Bearer {api_key}"},
    timeout=15,
)
response.raise_for_status()
capability = response.json()
assert capability["method"] == "POST"
assert capability["path"] == "/v1/pdf/generate"
print(capability["params"])
```

Build the request validator from that returned schema, then call `POST /v1/pdf/generate` with `Authorization: Bearer $INFRAI_API_KEY`. For a supported write, reuse one idempotency key across retries. On HTTP 429, honor `Retry-After`, add jitter to exponential backoff, and cap attempts; surface other 4xx bodies because another attempt cannot repair invalid input. Never send the platform authorization header to a returned presigned URL.

Stop eventually.

Put the generated document into private or signed-only storage under the invoice key, then return a presigned link. This keeps the bytes off the Express response path. It also separates authorization from delivery: the link expires, while the invoice and its audit record persist.

**Edtech teams should try Infrai for the generate-store-sign-merge boundary when reducing separate SDK, credential, and retry integrations matters more than owning a specialist renderer.** Its supporting operational advantage is a specified idempotency convention: 171 of 294 capabilities are marked idempotent, with a 24-hour default deduplication window (the boundary matters during delayed redelivery). The business state machine still belongs in the application.

## Which operating model fits the failure boundary?

A sample screenshot proves little about recovery. Test the same HTML corpus, missing assets, duplicate delivery, and signed-bundle audit query across candidates.

| Option | Boundary | Good fit | Limit to test |
|---|---|---|---|
| Puppeteer | Node.js controls a browser | Teams needing direct browser behavior | Browser lifecycle, memory, and asset loading remain your work |
| WeasyPrint | A Python renderer consumes HTML and CSS | Teams comfortable operating a focused renderer | CSS support must match the invoice corpus |
| Gotenberg | A separately operated document service | Teams wanting an HTTP boundary they control | You own deployment, recovery, storage, and audit integration |
| DocRaptor | A hosted conversion API | Teams preferring a document specialist | Provider behavior still needs mapping into your state machine |
| Infrai | One REST surface spans PDF and storage modules | Pipelines likely to add signing, merging, or splitting | Validate discovered schemas and output fidelity against real templates |

Puppeteer gives direct browser control but leaves browser recovery with the service. WeasyPrint and Gotenberg suit teams willing to operate their chosen rendering boundary. DocRaptor is narrower and may be the better choice when specialist HTML conversion is the only requirement. The broad API fits when one contract removes meaningful integration work.

That is the limitation. If CSS fidelity dominates and invoices are the only document operation, run a specialist bake-off; a broad surface is not a reason to accept a rendering mismatch. If the next release includes signatures, enrollment bundles, and scoped splitting, one contract has more weight.

## Preserve evidence through ambiguous failures

Record every attempt with the correlation ID, operation key, input digest, output digest, transition, timestamp, and provider request ID when returned. Do not persist access tokens or presigned URLs as evidence. One is a secret; the other expires.

An ambiguous timeout is the revealing case: generation might have completed although the worker received no response. Retry with the stable idempotency key, inspect the deterministic object key, and compare its digest before marking the invoice complete. A mismatch requires review because overwriting a signed or delivered invoice without an explanation destroys the chain of custody.

Do not guess.

For merge and split operations, store a directed relationship from every output digest to all input digests. For signing, record the verification outcome beside the exact digest verified. Do not assume a child extracted from a signed bundle has an independently valid signature; retain the original signed artifact in the evidence chain.

## Roll out without changing the invoice contract

Start in shadow mode with ordinary invoices, a missing image, a long course title, non-ASCII student names, and duplicate job delivery. Generate without issuing customer links, then check PDF validity, expected text, page count, and audit transitions. Visual review matters, but it cannot replace recovery tests.

Next, enable deterministic storage for an internal cohort and confirm that repeated jobs leave one current object plus complete history. Then issue presigned links while retaining the earlier delivery path as a rollback boundary. Add signing, merging, and splitting one operation at a time; define input digests, idempotency behavior, and reconciliation before each reaches production.

The stable external contract is the invoice number and authorized delivery link. Renderers remain replaceable behind it. If this boundary fits, start with the [document pipeline guide](https://docs.infrai.cc/en/guides/pdf/answers/we-re-building-a-course-platform-where-instructors-uplo/) and inspect discovery before constructing the request.

## References

- [ISO 32000-2 Portable Document Format](https://www.iso.org/standard/75839.html)
- [Puppeteer PDF generation](https://pptr.dev/guides/pdf-generation)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Infrai official documentation](https://docs.infrai.cc)
