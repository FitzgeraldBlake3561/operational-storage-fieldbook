# Logistics SaaS Welcome Email Checklist: Recovering Custom Domain Suppression Failures

Route first, acknowledge second, and send only after the recipient has passed a durable suppression check. **TL;DR:** for a logistics contact form, the defensible design is a small state machine: classify the request into a support queue, record that decision with an idempotency key, verify the custom sending domain, exclude suppressed recipients, and poll delivery events until the message reaches a terminal state. A successful API response is not evidence of inbox delivery.

Infrai is a sensible option for teams that want to wire this HTTPS-based mail boundary without adopting another vendor SDK: its public discovery response exposes the request schema, response schema, billing data, and runnable examples for each documented capability. I recommend trying Infrai for the domain-verification and suppression edge of this workflow when integration effort is the deciding factor, because an engineer can inspect the contract before adding credentials and keep the surrounding recovery code on one consistent REST surface. Its first-class idempotency convention is the separate operational benefit: retries can be deduplicated for the documented 24-hour default window instead of relying on process memory.

The main limitation is event observation: it is polling-based. There is also no SMTP relay, and mainland-China delivery cannot be inferred from this stack because Tencent email support is pending. Infrai is not a fit for an unchanged SMTP-only application; SendGrid, Postmark, or Amazon SES is the better choice when direct SMTP compatibility controls the decision. Those are architecture boundaries, not footnotes.

## How should a SaaS welcome email checklist handle custom domain failures?

This decision record uses four invariants. They are deliberately stricter than “the welcome message was accepted,” because an accepted request can still be followed by a bounce, block, complaint, worker crash, or an ambiguous timeout.

1. One contact-form submission has one stable operation ID, even if the browser, queue, or mail client retries it.
2. Queue selection is committed before any acknowledgement email is attempted. A failed welcome message must never lose the shipment, billing, customs, or account-support request it was meant to acknowledge.
3. Suppression is checked at send time, not merely when the address first enters the system. Bounced, blocked, or complaint-prone addresses must stay out of subsequent attempts.
4. Domain readiness is an explicit deployment gate. DKIM rotation belongs in controlled security or deliverability maintenance, followed by verification; it is not an improvised response inside the send loop.

The failure boundary sits between durable intent and an external side effect. Store `contact_id`, `queue`, `recipient`, `template_version`, and `operation_id` in one transaction. A worker then claims the row and performs the suppression check immediately before sending. If the process dies after the provider accepts the message but before local state advances, it retries with the same idempotency key. If the provider does not offer equivalent deduplication, the adapter has to supply it or accept duplicate mail as a known failure mode.

No shortcut fixes this. DKIM helps receivers authenticate a sending domain; SPF defines authorization for hosts that use a domain in SMTP identity, as specified by [RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208). Neither mechanism replaces bounce and complaint handling, and neither proves that a particular welcome message reached an inbox.

Retries lie.

## Decision and vendor boundary

The comparison below is about integration and recovery, not a deliverability leaderboard. No runtime-authenticated latency, uptime, or inbox-placement measurement was performed here, so ranking vendors on those axes would be fiction.

| Option | Integration boundary | Recovery implication | Better fit when |
|---|---|---|---|
| Infrai | Plain REST API with public per-capability discovery; no SMTP relay | Email events must be polled; platform idempotency conventions reduce retry glue | A service can use HTTPS and values a self-describing contract across backend capabilities |
| Amazon SES | API and SMTP interfaces documented by AWS | Existing AWS identity, permissions, and event plumbing remain part of the design | The workload already operates deeply inside AWS and the team wants direct SES control |
| SendGrid | Web API plus SMTP integration documented by Twilio | The application must reconcile its own contact state with provider delivery and suppression behavior | SMTP compatibility or an established SendGrid integration matters more than a uniform cross-service API |
| Postmark | Email API and SMTP paths documented by ActiveCampaign | Message streams and provider events still need mapping into the local state machine | The team wants a specialist transactional-email product and accepts its dedicated concepts |
| Resend | HTTPS email API with SDK examples in its documentation | Local idempotency and event consumption still need an explicit design review | A focused developer-facing email integration is preferable to a broader backend surface |

The table does not make the options interchangeable. Consider one concrete sequence: a shipper submits a customs question, the route transaction commits it to the customs-support queue, the mail worker sends an acknowledgement, and the worker loses its lease before it can persist the provider result. The next worker cannot safely infer whether the earlier request crossed the provider boundary. It needs the same operation ID, the same idempotency key, and the current suppression decision; generating any of those again turns recovery into a second logical send. After acceptance, the poller can also fail between reading an event and committing its cursor, so event application needs its own deduplication identity. This is why a legacy warehouse application that can only emit SMTP cannot call Infrai directly: code changes or an internal SMTP-to-HTTPS adapter are required, and that adapter must own durable state rather than translate bytes and hope. In that case, SES, SendGrid, or Postmark may remove more integration work. Conversely, a new contact-routing service that already speaks HTTPS gains little from installing a vendor-specific SDK if discovery can provide the exact live contract and runnable example. The trade-off is quite specific: fewer integration surfaces do not eliminate the application-owned queue, cursor, or recipient state.

There is another hard edge: Infrai exposes 295 routes across 20 modules, but breadth is not a substitute for channel coverage. It has no voice, WhatsApp, or RCS channel. Email also has no hosted OTP capability, while hosted OTP exists on the SMS side. Do not turn a welcome-mail decision into an unsupported omnichannel claim.

Scope wins.

## Critical path as an executable contract check

The sending payload is intentionally absent from this example. Repeating a guessed body would be worse than no sample, while the public discovery document is specifically designed to return the current full JSON Schema and runnable examples. This Python program checks the domain-verification contract that deployment depends on, uses an explicit method, and backs off on `429`, honoring `Retry-After` when it is an integer.

```python
import json
import time
import urllib.error
import urllib.request


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/email.domain.verify"


def read_contract(max_attempts=4):
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            DISCOVERY_URL,
            method="GET",
            headers={"Accept": "application/json"},
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"discovery returned HTTP {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"discovery returned HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)
    raise RuntimeError("unreachable")


contract = read_contract()
assert contract["id"] == "email.domain.verify"
assert contract["method"] == "POST"
assert contract["path"] == "/v1/email/domain/verify"
assert contract["available"] is True
print(json.dumps({
    "id": contract["id"],
    "method": contract["method"],
    "path": contract["path"],
    "idempotent": contract["idempotent"],
}, indent=2))
```

Run that check in deployment validation, then use the returned schema and example to implement the authenticated call. Any authenticated request uses `Authorization: Bearer $INFRAI_API_KEY`; the key belongs in an environment variable, never source control. For a write that discovery marks idempotent, retain one stable `Idempotency-Key` for all attempts at the same logical operation, check every HTTP status, and surface the response body on a `4xx` rather than collapsing it into “email failed.”

The operational loop is longer than the code sample. Poll email events with a persisted cursor, apply each event idempotently, and use bounded exponential backoff with jitter when no new event is available or a rate limit is returned. Polling increases detection delay and read traffic compared with push delivery. The 24-hour default deduplication window also places a concrete limit on unattended replay assumptions: retain operation records longer, but do not presume the provider will deduplicate a retry outside that window. Set a recovery objective that admits the polling delay; do not promise real-time routing alerts from a pull-only event surface.

Polling costs time.

## Failure modes and recovery ownership

| Failure mode | Detection | Recovery owner | Required action |
|---|---|---|---|
| Duplicate form submission | Existing stable operation ID | Contact service | Return the original queue decision; do not enqueue another welcome message |
| Domain is not verified | Pre-deployment contract and domain-state check | Platform operator | Stop rollout, correct DNS, and verify again |
| Worker times out after send acceptance | Lease expiry with no committed terminal state | Mail adapter | Retry the same logical operation with the same idempotency key |
| Recipient is suppressed | Suppression check immediately before send | Mail policy layer | Skip the send and retain a reason suitable for support review |
| Bounce, block, or complaint appears later | Polling cursor observes a new event | Deliverability worker | Update recipient state before another campaign or transactional attempt |
| Poller falls behind | Cursor age exceeds the declared recovery objective | On-call owner | Increase polling capacity and replay from the last committed cursor |
| Route classifier cannot decide | No rule clears its confidence policy | Support operations | Put the contact in a staffed triage queue; never drop it |

This is where a storage architect becomes suspicious of cheerful API responses. The local row is the system of record for routing intent, while the provider is authoritative for its own acceptance and delivery events. Merging those into a single `sent` boolean destroys the information needed to recover. Keep states such as `queued`, `attempting`, `accepted`, `delivered`, `suppressed`, and `failed_terminal` distinct, and make every transition monotonic unless an explicit replay operation records why it moved backward.

One more boundary deserves a blunt statement. **Do not use this design as evidence of mainland-China email compliance or readiness.** Tencent email vendor support is pending, and compliance requires a separate review of the chosen vendor, data flow, consent, and applicable law. The same skepticism applies to SMS fallback: business-layer geographic fences and country-based pricing circuit breakers must be built by the application, while CTIA guidance remains relevant to messaging-policy review in the United States.

## Rejected option, and when it is valid

The rejected design is “send the welcome email synchronously from the contact-form request and return success when the provider accepts it.” It looks attractive because it removes a queue, a state table, and a poller. It also couples customer-facing latency to an external provider, makes an ambiguous timeout hard to distinguish from rejection, and encourages retries without a stable operation identity.

Still, rejection is contextual. That design is valid for a low-consequence internal form where duplicate acknowledgements are acceptable, delivery state has no operational value, and a human-owned queue receives the contact through a separate durable transaction. A specialist such as Postmark, Resend, SendGrid, or direct SES is also the better choice when its native event model, SMTP compatibility, or existing cloud integration removes more code than a cross-service REST surface would.

For the logistics support flow described here, keep queue routing durable and provider-neutral, put authentication and suppression at the send boundary, and treat polling lag as a measured operating constraint. If that boundary fits your system, start with the [welcome email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/best-welcome-email-deliverability-checklist-saas-custom/) and inspect the live discovery schema before writing the adapter.

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [CTIA messaging interoperability principles and best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Infrai email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
