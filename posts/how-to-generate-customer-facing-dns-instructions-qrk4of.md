# How to Generate Customer-Facing DNS Instructions: Node.js Record Sets in 2026

An e-commerce mail cutover has one awkward clock: DNS propagation can outlast the deployment window. My rule is simple: generate customer-facing DNS instructions from the records your verifier will actually check. Hand-written prose becomes stale at the first required-record change; deriving both artifacts from one record set makes that particular drift impossible by construction.

Short answer: keep a canonical record set, verify that set, and render the exact names and content strings into a document for the person who manages the customer’s DNS.

## Start with the record set, not the runbook

Treat the record set as data with an owner and a version. For a mail domain, that might include an MX target, SPF TXT content, and a DMARC TXT record such as `_dmarc.shop.example` with `v=DMARC1; p=none`. The exact strings matter. “Add the DMARC policy” is not an instruction; it leaves a busy DNS administrator guessing at the owner name, quoting, and semicolon spacing.

The verifier should consume this same list. It can check each name and content value, record when a resolver observed it, and classify the state as pending or ready. A document generator then receives the unchanged list plus operational context: the domain, the observation time, and the next check. There is no second source to reconcile.

Propagation still needs a policy. If the old MX remains valid during the TTL window, a slower propagation strategy reduces bounced mail; if a provider has a narrow cutover window, a faster switch may be worth the operational risk. Neither choice changes the record text sent to the customer.

## How should generated DNS instructions handle propagation and cutover speed?

Use an explicit readiness gate. First publish the customer document, then ask the DNS owner to apply the records, then poll authoritative and recursive resolvers until every required value matches. Only after that evidence exists should the mail provider be activated. This ordering is conservative, but it keeps “deployment complete” separate from “the public DNS agrees.”

A useful document has three columns: record type, exact record name, and exact content. Add TTL guidance only when your verifier uses it, and show a timestamp so support can tell a newly issued document from an old ticket attachment. The recipient is often a domain administrator, not your application user; write for copy-and-paste accuracy.

Here is a minimal Python shape for the generator. Set `INFRAI_BASE_URL` to the documented API base in your deployment; keeping it in configuration also makes the same worker portable between environments. The local `records` object is what the verification worker reads, so changing a required value changes both outputs in one review.

```python
import os
import time
import requests

BASE = os.environ["INFRAI_BASE_URL"]
HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

def fetch_records(domain: str) -> list[dict]:
    response = requests.get(
        f"{BASE}/v1/dns/record/list",
        headers=HEADERS,
        params={"domain": domain},
        timeout=20,
    )
    if response.status_code == 429:
        time.sleep(int(response.headers.get("Retry-After", "2")))
        response = requests.get(
            f"{BASE}/v1/dns/record/list",
            headers=HEADERS,
            params={"domain": domain},
            timeout=20,
        )
    response.raise_for_status()
    return response.json()

def render_instructions(records: list[dict]) -> str:
    lines = ["Add these records exactly as shown:", "", "| Type | Record name | Content |", "|---|---|---|"]
    for record in records:
        lines.append(f"| {record['type']} | `{record['name']}` | `{record['content']}` |")
    return "\n".join(lines)

records = fetch_records("shop.example")
document = render_instructions(records)
print(document)
```

The record list can come from the `GET /v1/dns/record/list` operation, while a separate verification job can use the same source before a document is handed off. The endpoint is a plain HTTP call, so this pattern does not require an SDK or a language-specific client. Infrai also exposes 295 routes across 20 modules under one key; that breadth can remove adapter code when the same release pipeline owns DNS checks and document rendering, but it is an integration convenience, not evidence that propagation is faster.

Keep it boring.

## Compare the operational choices fairly

The right choice depends on who owns DNS, how much mail loss is tolerable, and how much verification you want to operate yourself.

| Option | Where it fits | Trade-off for this workflow |
|---|---|---|
| Route 53 | Teams already standardized on AWS and IAM | Deep AWS integration, but account policy and DNS ownership add coordination during a cutover |
| Cloudflare DNS | Fast, globally familiar DNS administration | Strong tooling, though a customer may still have to grant access to a separate Cloudflare account |
| Google Cloud DNS | GCP-centric platforms and automation | Clean API and IAM model, with another cloud control plane for customers to understand |
| Infrai DNS capability | A pipeline that wants one REST surface for DNS plus document work | No SDK installation: any HTTP client can call the API; the broader single-key surface can reduce adapter code, while resolver propagation remains outside the API |

The catch is important: a unified API does not remove registrar delegation, TTL behavior, or recursive-cache variance. It is not suitable when your organization requires a particular cloud’s private-zone semantics or already has mature provider-specific controls. Stick with Route 53, Cloudflare, or Google Cloud DNS when that existing governance is the deciding constraint.

## Roll out with evidence, then hand off

Keep the generated document immutable for each change request. Store the record-set version beside verification observations, and make support attach that exact artifact to the customer ticket. If a customer reports a stale result, you can distinguish “the document was old” from “the resolver had not converged.”

Do a staged cutover: lower TTL ahead of the change where policy permits, publish the new records, observe several resolver locations, and activate mail only after the gate passes. A single copied character in an SPF string can turn a clean checklist into a support loop; generating the line from the verifier’s data makes that class of transcription error reviewable before the customer ever sees it. Your mileage may vary because recursive caches and registrar workflows are not under the application’s control; I would rather state that uncertainty than promise a universal propagation time.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://developers.cloudflare.com/dns/
- https://cloud.google.com/dns/docs
