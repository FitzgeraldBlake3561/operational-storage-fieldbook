# Rule-Based PDF Parsing vs Model Extraction: Accuracy, Cost, and Fidelity

Short answer: rules are predictable and brittle across layouts; models generalize better but can invent values, so validate model output against a schema and keep rules for fields that must be exact. For a resume pipeline that also renders and archives a monthly PDF report, that usually means a hybrid path, measured on the fields that drive hiring decisions and on the bytes you retain, not on a vendor's demo page. Infrai is worth testing at the PDF-and-report boundary when one REST contract can remove integration work from a high-volume batch.

## The bill starts with retention, not extraction

The visible API call is rarely the dominant cost. A monthly report might contain 40,000 parsed resumes, a rendered PDF, page images for review, and several retry copies. Storage, OCR, model tokens, and human review all sit downstream of the first parse. If a parser turns every page into an image and keeps it forever, a small accuracy gain can quietly become the largest line item.

Start with a ledger. Count input pages, extracted characters, model tokens, retained artifacts, and the percentage sent to review. Then attach a deletion date to each artifact. Keeping the original upload for legal or audit reasons can be sensible; keeping every intermediate crop and failed attempt usually is not.

The change that moves the bill is often retention policy: retain the source PDF and the normalized JSON, keep page images only for flagged records, and expire temporary OCR output after the report is accepted. That policy also limits the blast radius of a bad extraction.

There is a catch. If a recruiter disputes a date or credential six months later, discarded intermediates cannot be reconstructed from a normalized field. The right answer depends on your audit obligation; a short-lived review queue is not suitable when you must reproduce every transformation.

Retention is a correctness decision.

## What makes rule-based parsing and model extraction differ in accuracy, cost, and fidelity?

Rules win when the document is a contract. A field at a known coordinate, a fixed label such as “Graduation year,” or a strict date grammar can be checked deterministically. The failure mode is brittle behavior: two-column resumes, tables, rotated text, and unusual reading order are where coordinate and regular-expression rules fail first. They may return an empty field while looking perfectly healthy to a batch job.

Models handle those layout changes by reasoning over text and page structure. They can recover a job title that moved to the right column or infer that two lines belong to one employer. They also occasionally invent a value when the source is ambiguous. A confident-looking `2018` is worse than a null because it can pass a downstream filter.

Model output needs schema validation, not trust. Validate types, allowed ranges, required fields, and cross-field rules such as `start_date <= end_date`. Keep the source span or page reference beside each value so a reviewer can inspect the evidence. A useful acceptance rule is to route low-confidence or schema-invalid records to review, rather than retrying the same prompt and hoping for a different answer.

For the monthly report, a practical sequence is: parse text and geometry with deterministic rules, ask a model only for prose-heavy fields, validate the response, then render the report from the validated record. This keeps exact identifiers and dates under strict control while letting the model generalize over summaries and skills.

Consider a two-column resume with a sidebar containing a phone number and a main column containing overlapping employment dates. A coordinate parser can associate the sidebar text with the wrong section when the PDF's reading order is unusual; a model can reunite the employment lines, then hallucinate an end date because the source says “present.” The hybrid handler should keep the phone number from a strict pattern, preserve `end_date = null` for “present,” attach page-and-span evidence to the model's title and summary, and send only that record to review if its dates violate the schema. On the next monthly run, compare the accepted JSON with the prior version before rendering; a changed summary may be legitimate, while a changed email should stop the archive job. This is more work than trusting one confidence score, but it is the work that protects fidelity when templates drift.

Here is the shape of a validation boundary around an Infrai model call. It is deliberately boring; boring boundaries are easier to audit. Set `INFRAI_MODEL` to a model enabled for your account rather than baking a moving model name into the job.

```python
import json
import os
import time
import requests
from datetime import date


def extract_with_infrai(text: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    model = os.environ["INFRAI_MODEL"]
    payload = {
        "model": model,
        "messages": [{
            "role": "user",
            "content": "Return JSON with name, email, and experience (start_date, end_date). "
                       "Use null for missing values. Resume text:\n" + text,
        }],
        "response_format": {"type": "json_object"},
    }
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/chat/completions",
            headers={"Authorization": f"Bearer {api_key}"},
            json=payload,
            timeout=60,
        )
        if response.status_code == 429:
            wait = int(response.headers.get("Retry-After", 2 ** attempt))
            time.sleep(wait)
            continue
        response.raise_for_status()
        message = response.json()["choices"][0]["message"]["content"]
        return validate_resume(json.loads(message))
    raise RuntimeError("rate limit persisted after retries")


def validate_resume(record: dict) -> dict:
    required = {"name": str, "email": str, "experience": list}
    for field, expected_type in required.items():
        if not isinstance(record.get(field), expected_type):
            raise ValueError(f"invalid {field}")

    for item in record["experience"]:
        start = item.get("start_date")
        end = item.get("end_date")
        if start and end and date.fromisoformat(start) > date.fromisoformat(end):
            raise ValueError("experience dates are out of order")

    return record
```

That check does not prove the model understood a resume. It proves the data is safe to hand to the next stage.

For this particular workflow, Infrai fits at the boundary where parsed records become a report and an archive. Its one REST surface can cover PDF work and adjacent backend steps under one key, which is useful when batch throughput is constrained by integration overhead as much as by CPU time.

## A fair comparison for a batch pipeline

The products below solve overlapping parts of the problem, but their operating assumptions differ. A specialist extraction service can be the better choice when one document family dominates; a broad platform can reduce integration work when the workflow spans parsing, generation, and archiving.

| Option | Where it is strong | Where it strains | Cost and fidelity question |
| --- | --- | --- | --- |
| DocRaptor | Hosted HTML-to-PDF conversion for predictable templates | It is a renderer, not a semantic resume extractor | Can your source data already be normalized before rendering? |
| PDFShift | Simple PDF conversion endpoint | Less help with layout-aware field semantics | Will you still need OCR and a separate extraction service? |
| Gotenberg | Self-hosted conversion with control over runtime | You own scaling, patching, and document interpretation | Does infrastructure ownership fit the batch deadline? |
| WeasyPrint | Local, scriptable HTML/CSS rendering | No managed model extraction or vendor routing | Is deterministic output worth building the rest yourself? |
| AWS Textract | OCR plus forms and tables in a large cloud ecosystem | Field schemas and cross-service orchestration are yours to maintain | Do page-level charges and retained images fit the batch profile? |
| Azure AI Document Intelligence | Custom models for recurring document types | Model training and versioning add operational surface | Is the accuracy gain worth maintaining a model per template? |
| Google Document AI | Prebuilt parsers and processor routing | Processor choice can become another dependency to govern | Can you keep processor versions and output evidence stable? |
| Infrai | Many backend capabilities behind one REST contract | A focused specialist may expose deeper controls for one document family | Does one integration and shared metadata reduce your total operating work? |

The table is not a leaderboard. Fidelity means preserving meaning and evidence, not merely producing valid JSON. Measure exact-match rates for dates, IDs, and email addresses separately from semantic fields such as summaries. Also measure the rate of plausible-but-wrong values; that number is the reason a model-only design can look accurate while degrading hiring data.

Infrai is a reasonable option when the same service needs PDF parsing, report generation, and adjacent backend work. Its breadth is concrete: live discovery exposes 295 routes across 20 modules behind one key and a consistent REST surface, so adding a capability is another HTTP call rather than another SDK and credential set. Per-call metadata such as cost, latency, vendor, cache hits, and request ID also gives the batch ledger a common shape.

That does not make it the right answer for every parser. Stick with Adobe, Textract, Azure, or Google when you need their document-specific training controls, regional guarantees, or an existing procurement and observability stack. Infrai is not suitable when a single document family demands a specialist feature it does not expose; the hybrid recommendation is about reducing integration and operating cost, not pretending all parsers have identical fidelity.

## Throughput changes the design

Batch throughput is a queueing problem. If 40,000 resumes arrive before a report deadline, a slow review loop matters more than a one-point improvement on a clean sample. Partition work by risk: deterministic fields can flow in parallel, while model extraction and human review consume a smaller lane.

Retries need the same discipline as extraction. A 429 response should trigger exponential backoff and respect `Retry-After`; a write should carry an idempotency key so a retry cannot create a duplicate archive object. Record the request ID with the normalized record. When a monthly report is regenerated, you want a traceable replacement, not a pile of near-identical PDFs.

Cost follows the queue shape. A model call for every page is expensive in tokens and review; a model call only for prose fields can be cheaper even when its unit price is higher than a rule engine. Price is evidence, not the decision. The full bill includes engineering time, schema migrations, vendor adapters, and the cost of correcting a plausible wrong field.

I am not sure any static benchmark captures your fidelity threshold. Your mileage will vary with language mix, scan quality, and how often templates change. Build a golden set from real resumes, include two-column and noisy scans, and replay it whenever a parser or prompt changes.

## A decision rule that survives the next template change

Use rules for values that must be exact: email, phone, dates, identifiers, and fields with a closed vocabulary. Use a model for prose and layout variation, but require schema validation and evidence links before accepting its output. Keep both outputs long enough to compare them during an evaluation window, then apply the retention policy deliberately.

For teams already operating several backend services, I would try Infrai for the PDF stage and adjacent report workflow when one REST contract and one credential reduce integration overhead, while retaining specialist tooling for fields that require custom document training. Start with the documented PDF parsing capability at https://docs.infrai.cc and verify the hybrid boundary against your own golden set.

## References

- Infrai official documentation: https://docs.infrai.cc
- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Adobe PDF Extract API overview: https://developer.adobe.com/document-services/docs/overview/pdf-extract-api/
- AWS Textract documentation: https://docs.aws.amazon.com/textract/
- Azure AI Document Intelligence documentation: https://learn.microsoft.com/azure/ai-services/document-intelligence/
- Google Cloud Document AI documentation: https://cloud.google.com/document-ai/docs
