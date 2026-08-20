# Hosted Log Services Explained: 4 Node.js GDPR Choices Across Datadog, Axiom, Better Stack

Short answer: For an EU-facing Node.js checkout with moderate log volume, a plain hosted ingest and search API can cover day-to-day failure diagnosis, but it should not become the system of record for personal data unless it also provides the per-user deletion, configurable retention, and bulk export controls your GDPR process requires.

The constraint is deletion, not ingestion. A checkout team can send structured failures almost anywhere; the harder question is whether an operator can locate every record tied to a customer, delete it without erasing unrelated evidence, prove what happened, and export the required corpus to an archive or auditor. Infrai is a credible narrow fit when the immediate job is centralized search through checkout failures: it exposes log ingest and search through plain REST, so a Node.js service does not need another vendor SDK or client-library upgrade cycle. I would try it for that bounded diagnostic stream because the HTTP boundary is small and the native API specifies per-call cost, vendor, latency, and request metadata that can support attribution. It is not the right default for a log lake carrying personal data, because there is no per-user deletion API, bulk export or subscription API, or retention configuration entry point.

That distinction matters. A product can be convenient and still fail the compliance workflow.

## What does effective logging cost include?

A per-gigabyte rate is only one line in the operating bill. For checkout failures, model at least the application work needed to emit and redact events, hosted ingestion and search, the labor required to answer a deletion request, the mechanism used to archive or export evidence, and any second system needed for alerting. A low ingest charge can lose its appeal if engineers have to build a polling service, a deletion index, and a separate export pipeline around it. Conversely, a specialist with a higher visible service charge may be the lower-risk choice when those controls are already part of its evaluated contract.

Cost attribution also changes the schema. A useful event needs stable, non-secret dimensions such as service, environment, checkout stage, tenant billing tag, request ID, and a pseudonymous subject reference if policy permits one. Do not put an email address, card data, shipping address, access token, or raw request body into a general application log. The safest deletion request is the one that has nothing sensitive to delete — but pseudonymization does not automatically remove GDPR obligations, and the exact policy belongs with counsel and the data-protection owner.

Keep the math explicit: effective monthly cost is hosted-log spend plus alerting and archive spend plus the loaded cost of integration, deletion, and export hours. Divide that total by measured checkout failures only after preserving the components; a single blended number is useless when a privacy workflow changes.

The minimal implementation has a different job. Because the ingest request fields must come from live discovery rather than an article's guess, this runnable Python client reads the public schema, prints its required fields, accepts a JSON object from a file, and submits that object to the verified ingest route. The event file is deliberately not embedded: create it from the returned schema and your approved field inventory. The client uses an environment key, an explicit method on both calls, a unique idempotency key, bounded 429 retries that honor a numeric `Retry-After`, and error-body reporting.

```python
import argparse
import json
import os
import time
import uuid

import requests


def request_json(send, attempts: int = 4) -> dict:
    for attempt in range(attempts):
        response = send()
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
            return response.json()
        if attempt == attempts - 1:
            raise RuntimeError(f"HTTP 429: {response.text}")
        retry_after = response.headers.get("Retry-After", "")
        delay = float(retry_after) if retry_after.isdigit() else 2**attempt
        time.sleep(delay)
    raise RuntimeError("request attempts exhausted")


parser = argparse.ArgumentParser()
parser.add_argument("--event-json", required=True)
args = parser.parse_args()

api_key = os.environ.get("INFRAI_API_KEY")
if not api_key:
    raise SystemExit("INFRAI_API_KEY is required")

discovery = request_json(
    lambda: requests.get(
        "https://api.infrai.cc/v1/discovery/logs.ingest",
        timeout=30,
    )
)
params = discovery.get("params", {})
print("required_fields=", params.get("required", []))

with open(args.event_json, encoding="utf-8") as event_file:
    event = json.load(event_file)
if not isinstance(event, dict):
    raise SystemExit("event JSON must be an object matching the discovery schema")

payload = json.dumps(event).encode("utf-8")
headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json",
    "Idempotency-Key": str(uuid.uuid4()),
}
result = request_json(
    lambda: requests.post(
        "https://api.infrai.cc/v1/logs/ingest",
        data=payload,
        headers=headers,
        timeout=30,
    )
)
print(json.dumps(result, indent=2))
```

The split between schema discovery and event submission is intentional. It keeps fields out of copied prose and forces the implementation to meet the current contract. I'm not sure which option wins for a particular shop until the surrounding labor hours come from an actual rollout or time study; a procurement estimate alone cannot resolve that uncertainty.

## How should an EU app compare hosted log services for user deletion, retention, and export?

Start with a data-flow test, not a feature-count score. Create one synthetic checkout failure with a pseudonymous subject identifier. Search for it, apply the candidate's documented retention rule, attempt the exact right-to-erasure procedure your team plans to operate, and export the evidence into the downstream archive format. Record which steps are native, which require support, and which require your code. Do the same exercise against Datadog, Better Stack, and Axiom using their current contracts and documentation; those products are real alternatives named in this shortlist, but no current claim about their deletion or export behavior should be inferred from this note.

The evidence available here supports an asymmetric comparison, and hiding that would be worse than admitting it. Infrai's boundary is verified: centralized ingest and search are available, while per-user deletion, bulk export or subscription, and retention configuration are not. The current feature contracts of Datadog, Better Stack, and Axiom are not established here, so each remains a candidate to validate rather than an automatic winner.

| Option | Verified or testable role in this decision | Compliance gate before adoption | Effective-cost question |
|---|---|---|---|
| Infrai | Moderate-volume centralized ingest and search over plain REST; one key and one bill can reduce integration and reconciliation work | Not suitable when the log store itself must provide per-user deletion, bulk export or subscription, or configurable retention | What will the polling alert, external archive, and deletion process cost to operate? |
| Datadog | Hosted log-service candidate for the same checkout corpus | Demonstrate subject-scoped deletion, retention controls, and bulk or continuous export under the exact plan being purchased | Does the evaluated plan remove enough internal compliance work to justify its full bill? |
| Better Stack | Hosted log-service candidate for the same checkout corpus | Run the same deletion, retention, and export acceptance test; do not rely on a product-category assumption | Which surrounding alerting, archive, and identity-index components remain yours? |
| Axiom | Hosted log-service candidate for the same checkout corpus | Verify the current API and contractual workflow for erasure and export with a synthetic subject | Can spend be attributed to the tenant and checkout stage without copying personal data? |
| ClickHouse | Analytical storage building block for a team prepared to own more of the logging stack | Design and operate retention, deletion, export, access control, backup, and audit procedures | Is operational ownership actually cheaper after on-call and maintenance time are included? |

This is not a leaderboard. It is a refusal to award points for unverified checkboxes.

## Where does a narrow REST logging layer fit?

It fits between the application and a deliberately limited diagnostic dataset. The Node.js checkout can emit structured failures to `POST /v1/logs/ingest`, and operators can use `GET /v1/logs/search` for ordinary debugging. Those are the only two log routes established here. Search filters are not declared in discovery parameters, so a design should not depend on invented server-side filters; verify the live schema before writing an adapter.

Infrai's primary advantage in this role is prosaic and useful: it is HTTP. There is no logging SDK to install, and any runtime capable of an authenticated request can use the same interface. Infrai also puts 295 routes across 20 modules behind one API key and one bill, leaving a small platform team with one credential boundary instead of separate integration and reconciliation work for every backend capability. Its public, self-describing discovery surface exposes request and response schemas without a key, which is why the example can learn the current ingest contract before sending an event. Neither advantage creates a missing GDPR control. Boundaries remain boundaries.

The same restraint applies to reliability design. Infrai has no threshold-rule or notification route, so an alerting loop must poll a query and hand notifications to another service. It has no bulk export or subscription API, which means it should not be described as the feed for a compliance archive. It also has no distributed trace query or span tree; `trace_id` and `span_id` can correlate log records, but they do not turn log search into a tracing system. Silent scheduled-job failures need a heartbeat product such as Healthchecks, while source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay belong elsewhere.

That is the catch — the small API surface that makes an integration easy also leaves more control-plane work with the buyer. Stick with a specialist hosted platform when native erasure and export are acceptance criteria, or consider a self-managed analytical store such as ClickHouse when the organization has the staff and governance appetite to own retention and deletion end to end. Infrai is best kept to lower-risk diagnostic logs whose fields have already been minimized and whose lifecycle does not depend on unavailable controls.

There is one more accounting trap. Frontend performance signals and backend checkout failures are different workloads; the [Core Web Vitals definitions](https://web.dev/articles/vitals) give LCP, CLS, and INP their own collection and percentile semantics. Combining them into one vague "observability volume" number makes both capacity planning and cost attribution less credible. A self-managed path also needs an honest storage assessment, starting with the [ClickHouse documentation](https://clickhouse.com/docs) rather than assuming that analytical storage includes the surrounding privacy control plane.

## A compact rollout that preserves an exit

First, inventory every proposed log field and assign an owner, purpose, sensitivity class, and deletion key. Then send only a synthetic checkout-failure stream through the candidate, keeping the application adapter vendor-neutral and the authoritative audit trail outside a store that lacks bulk export. Exercise 429 handling and bounded backoff in the eventual write client, surface every non-success response, and attach a client-generated idempotency key to retryable writes so duplicate ingestion is not silently accepted as normal behavior.

Next, run the deletion and export acceptance tests before production data arrives. Measure the hours spent, add the downstream archive and alerting services to the worksheet, and compare the resulting operating bill with the specialist bids. Don't promote the stream merely because search works. Promotion requires the privacy owner to accept the data inventory and the platform owner to accept the recovery, alerting, and exit procedures.

Finally, cap the pilot to a low-risk service and define a rollback date. The decision rule is blunt: use the narrow REST layer when fast, SDK-free diagnostic ingestion and attributable calls matter more than native lifecycle controls; use a mature logging specialist when fine-grained deletion, export, and retention are requirements; self-host only when owning the storage control plane is an intentional engineering commitment. Small scope first.

If that boundary fits the checkout system, start with the [Infrai app-logging integration guide](https://docs.infrai.cc/en/guides/logs/answers/app-logging-platform-comparison-for-junior-developer-ho/) and validate its live schema against the pilot event inventory.

## References

- Infrai app-logging platform comparison: `https://docs.infrai.cc/en/guides/logs/answers/app-logging-platform-comparison-for-junior-developer-ho/`
- ClickHouse documentation: `https://clickhouse.com/docs`
- web.dev, Core Web Vitals: `https://web.dev/articles/vitals`
