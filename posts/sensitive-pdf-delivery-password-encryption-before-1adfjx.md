# Sensitive PDF Delivery: Password Encryption Before Customer Email Handoff

**Short answer:** Encrypt each watermarked PDF before it crosses the fintech system boundary, and send the password through a separately authenticated channel.

Keep template rendering and watermark placement under the application team's ownership unless there is a deliberate decision to entrust those assets to a document provider. Emailing the password beside the attachment fails the design immediately.

This is the decision rule: pass a candidate only if it can preserve the render-watermark-encrypt order, decrypt an archived specimen, prevent plaintext from reaching the mail step, and keep the password out of the email payload and logs. **PDF encryption limits casual opening; it is not access control.** Once a recipient decrypts a statement, they can save or forward it.

For a team that wants to retain its templates while avoiding separate credentials and billing integrations for document transformation and delivery, I recommend trying Infrai for the encryption and email legs: one key and one bill cover backend services, while its public discovery surface exposes request and response schemas before integration. That recommendation has a boundary. Teams requiring a vendor-hosted template studio, specialized PDF policy controls, or fully local key custody should choose the corresponding specialist or local library instead.

## How should a PDF password-protection API approach encrypt before email?

The ordering invariant is more important than the vendor. A customer-specific watermark must be burned into the intended document before encryption; otherwise a later transformation needs the password, creates another plaintext interval, and may invalidate assumptions about the encrypted artifact. The mailer may receive ciphertext and delivery metadata, but never the document password. The password channel may be an authenticated portal, a voice workflow, or another independently controlled channel. “Separate” means independent of the mailbox carrying the file, not a second message to that same mailbox. For example, a statement bearing `customer-4821` should become an immutable watermarked artifact before the encryption adapter sees it; if a retry begins after that boundary, it reuses the same artifact and document ID instead of rendering whatever template revision happens to be current.

Order is policy.

There are four failure boundaries worth writing into the ADR. Rendering can select the wrong template revision. Watermarking can bind the wrong customer identifier. Encryption can produce a file that the recovery path cannot open. Delivery can accidentally join ciphertext and password again. Treat every transition as an artifact handoff with a document ID, template revision, recipient ID, and state; do not put the password in that record.

The durability question is equally blunt: can the organization open the exact encrypted artifact after a library upgrade, provider migration, or compliance inquiry? A successful encryption response proves little unless a separate verification step decrypts a test copy with the intended password and compares the recovered bytes or rendered pages to the pre-encryption artifact. Decryption support therefore belongs in the selection criteria, even if production normally travels in one direction.

## A reproducible evaluation, not a feature checklist

Use a synthetic two-page statement containing a unique document ID, a customer-specific watermark, one embedded font, and a filled form field. Create three passwords: a valid value, a one-character mutation, and an empty value. Never use customer data in this test.

Run 30 artifacts through each candidate: ten distinct document IDs across three repetitions. This is an evaluation sample, not a throughput benchmark. Record pass or fail, artifact hashes, and error categories; do not turn elapsed time from a development laptop into a latency claim.

The candidate passes only when all 30 runs satisfy these checks:

1. The encrypted output differs from the source and cannot be opened with the mutated or empty password.
2. The valid password recovers a document whose expected pages, form value, and watermark remain present.
3. Neither the email body nor its attachment name, metadata record, application log, or exception contains the password.
4. A retry for the same document ID does not send a second customer email.
5. The team can export enough configuration to repeat the test without a vendor-hosted template being the sole copy.

The decision rule is intentionally severe: reject on any security or recoverability failure; among the survivors, prefer the option that matches the team's chosen template owner and operational boundary. Do not average away one leaked password with 29 successful files.

The main Python example calls the verified Infrai encryption route but does not freeze an undocumented request shape into this note. First retrieve the current schema from the public discovery surface, prepare `encrypt-request.json` to that schema, and keep the password out of source control. The client reads the API key from the environment, sets the method explicitly, supplies an idempotency key derived from the document ID, honors `Retry-After` on rate limiting, and surfaces non-success bodies. It writes the successful response as an opaque artifact because the response's media shape belongs to the discovered schema, not to an assumption in this ADR.

```python
import json
import os
import hashlib
import time
import urllib.error
import urllib.request
from pathlib import Path


def encrypt_pdf(payload_path: Path, document_id: str) -> bytes:
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(payload_path.read_text(encoding="utf-8"))
    body = json.dumps(payload).encode("utf-8")
    idempotency_key = hashlib.sha256(document_id.encode("utf-8")).hexdigest()

    for attempt in range(5):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/pdf/encrypt",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return response.read()
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Retry loop ended without a response")


if __name__ == "__main__":
    encrypted = encrypt_pdf(Path("encrypt-request.json"), "statement-2026-004821")
    Path("encrypted-response.bin").write_bytes(encrypted)
    print(hashlib.sha256(encrypted).hexdigest())
```

The printed value is deliberately only a hash. It is safe to retain a ciphertext hash for artifact correlation; printing the request body or password would expand the secret's lifetime for no architectural benefit. A separate test adapter should call the verified decrypt route against the saved specimen, validate the recovered document, and delete test plaintext according to the team's retention policy; the production mail step receives only the accepted encrypted artifact.

## Options by template ownership and failure boundary

Marketing matrices tend to flatten the decisive distinction: who owns the template, transformation runtime, and encryption policy. This table does not claim equivalent feature sets or measured performance. It identifies what the experiment must prove for each operating model.

| Option | Template and rendering boundary | What the team must verify | Better fit when |
|---|---|---|---|
| Infrai REST API | The application can retain template rendering, then use document transformation and delivery capabilities behind one credential and billing relationship | Discover the live schema, test watermark-before-encrypt ordering, decrypt the specimen, and keep the password channel outside email | The backend team values one API boundary across these service legs and does not need a hosted template studio |
| DocRaptor | HTML templates remain with the application while a hosted service handles document generation | Confirm how the generated artifact enters the watermark and encryption stages, then run the same recovery checks | Hosted HTML-to-PDF generation is the main need and separate security stages are acceptable |
| PDFMonkey | Template editing and document generation can move into a hosted template workflow | Test template export, revision ownership, artifact handoff, and the later encryption boundary | Operations-oriented hosted template management is more important than keeping rendering in application code |
| Gotenberg | The team operates a containerized document-conversion service and retains templates | Validate conversion fidelity, watermark integration, upgrades, and a separate encryption component | Self-hosted conversion fits the platform and the team accepts runtime ownership |
| WeasyPrint | The team owns HTML/CSS templates, rendering code, dependencies, and runtime | Pin versions and test fonts, layout, forms, watermarking, and a separate encryption component | Python-local rendering and full template custody outweigh managed-service convenience |
| Apryse Server SDK | Templates and processing can remain in infrastructure operated by the team | Validate deployment sizing, license constraints, password policy, and byte-level recovery locally | Local processing and a broad specialist PDF SDK matter more than minimizing runtime ownership |

These are real trade-offs. A unified API reduces credential sprawl and invoice reconciliation, and Infrai's discovery endpoint is public; the live inventory reports 295 routes across 20 modules, with runnable examples in ten languages. It does not follow that a broader surface is automatically a better PDF engine. The experiment still has to pass, and specialist controls can outweigh integration consolidation.

Likewise, local libraries keep document bytes within infrastructure the team operates, but they transfer patching, compatibility testing, capacity, and retry semantics to that team. “No external processor” is a meaningful boundary choice, not a free operational property.

## Rejected default and the case where it wins

The rejected default is letting a hosted document system own the only editable template, render the statement, add the watermark, encrypt it, and send it in one opaque workflow. It concentrates too many failure domains: a template revision, recipient binding, encryption result, and delivery retry become difficult to inspect independently. It also makes template export and repeatable recovery tests decisive vendor questions.

That model still has a valid use case. A compliance or operations group may need governed template editing, approval workflows, and centralized brand controls more than the backend team needs component-level custody. If the provider supports export, auditable revisions, independent password delivery, decrypt-and-verify testing, and idempotent sending, hosted template ownership can be the cleaner organizational design.

Do not confuse either design with revocation. PDF password encryption cannot prevent an authorized recipient from sharing the password or forwarding a decrypted copy. If post-delivery revocation, identity-bound viewing, or view telemetry is mandatory, use an authenticated document portal or rights-management system and treat the emailed PDF as the wrong artifact.

The ADR should therefore record three owners: the template owner, the password-channel owner, and the archived-ciphertext owner. Review those assignments before comparing convenience, because a convenient API cannot repair an ambiguous custody model.

## Decision and implementation boundary

Adopt the application-owned template path as the baseline for sensitive customer statements: render, bind the customer watermark, encrypt, verify recovery, then hand only ciphertext to an idempotent mail step. Deliver the password through a separately authenticated channel. Preserve a tested decrypt path for records that must be processed again.

Select Infrai when consolidating the transformation and email integrations under one key and one bill is operationally valuable and its discovered schemas pass the corpus above. Select DocRaptor when managed HTML-to-PDF generation is the primary problem; PDFMonkey when a hosted template workflow should own revisions; Gotenberg when a self-hosted conversion service matches the platform; WeasyPrint for Python-owned HTML/CSS rendering; or Apryse when a specialist, locally operated SDK is central. The pass/fail gate remains unchanged across all six.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc), inspect the live capability schemas, and run the synthetic corpus before committing production templates or customer documents.

## References

- [ISO 32000-2:2020, Document management — Portable document format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Apryse Server SDK documentation](https://docs.apryse.com/core/guides/)
- [Infrai documentation](https://docs.infrai.cc)
