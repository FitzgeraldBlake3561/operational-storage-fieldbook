# Short DNS TTLs Everywhere or Pre-Change Reduction (Clinical Mail Chooses Planning)

A healthtech sender cannot treat DNS agility as the only objective. SPF, DKIM, and DMARC records sit on the path to mail acceptance, while resolvers cache answers to avoid repeating resolution work. **Use planned TTL reduction before a controlled authentication change, then restore the normal TTL after caches have aged out.** Permanent short TTLs are justified only when a tested need for unplanned switching outweighs greater dependence on authoritative DNS availability, resolution latency, and query volume.

TL;DR: keep the steady-state TTL long enough to provide useful caching, lower it at least one old-TTL interval before a scheduled change, verify the new answer through independent recursive resolvers, make the change, and leave the reduced value in place until old data can no longer remain cached. For clinical mail, successful authentication and DMARC reporting are evidence. A fast control-plane update is not.

## Should short DNS TTLs run everywhere or only before a planned change?

A TTL limits how long a caching resolver may retain an answer; it does not recall an answer already cached under the previous value. If a DMARC record was cached with a 24-hour TTL at 09:00 and an operator changes both the record and its TTL at 10:00, that resolver may continue serving the old record until 09:00 the next day. The new five-minute value cannot travel backward into that cache.

Resolvers do not form one synchronized global cache. A query from an office network, a CI runner, and a public recursive service can encounter different cache ages, while a negative answer has caching behavior of its own under RFC 2308. Deleting a selector or publishing a name for the first time therefore needs the same care as replacing an existing value.

That distinction matters.

For healthtech mail, a DNS answer is only an intermediate signal. The sender signs a message, the receiver retrieves the public key, SPF evaluates the authorized path, and DMARC checks identifier alignment and policy. Aggregate reports later expose how receivers evaluated real traffic. RFC 7489 defines that reporting, but delayed reports are not an immediate deployment probe.

## Derive the policy from the failure budget

Start with change modes. DKIM rotation can overlap old and new selectors, removing the need for a rushed destructive replacement. SPF changes can exclude legitimate senders if an authorization path is removed too early; RFC 7208 also limits terms that cause DNS queries to 10 during evaluation. DMARC policy changes affect receiver handling. None of those risks is repaired merely by asking caches to expire every few minutes.

The steady-state TTL should follow four inputs: authoritative-service availability, expected emergency recovery time, ordinary change frequency, and the longest stale-answer window the mail team can tolerate. There is no universal magic number in the standards. A 24-hour example explains cache timing; it is not a recommendation for every domain.

Preserve overlap whenever the protocol permits it. Publish a new DKIM selector before sending with it, retain the old selector while messages bearing the old signature can still be evaluated, and remove it after the agreed overlap period. For SPF, stage authorization additions before traffic moves and retire old paths afterward. For DMARC, validate syntax and reporting destinations before changing enforcement.

## Compare the operational evidence

| Decision factor | Permanently brief TTL | Planned reduction |
|---|---|---|
| Scheduled authentication work | Ready without advance scheduling | Requires waiting at least the previous TTL |
| Authoritative load | More cache misses over time | Increase is limited to the change window |
| DNS outage exposure | Cached answers protect clients for less time | Normal TTL retains a longer cache buffer |
| Emergency correction | Lower bound for newly fetched positive answers | Normal TTL may delay correction |
| Evidence | Low configuration value proves little | Rollout joins expiry, lookup checks, and authentication results |

The trade-off is asymmetric. Permanent brief TTLs buy readiness for an event that may never happen, every hour of every day. Planned reduction spends operator effort around known changes and provides worse emergency agility between them. A 300-second TTL everywhere can reduce the cache window versus a 24-hour value for newly fetched answers, but it cannot remove network latency, force a receiver to query immediately, or repair a malformed authentication policy. **For routine clinical-mail authentication, planned reduction wins because overlapping keys and staged authorization handle most changes more safely than rapid replacement.**

There is a limitation to that choice. Planned reduction is not suitable for a domain that must move without notice and cannot overlap records; if its authoritative service has demonstrated capacity and availability under the resulting miss rate, permanently short TTLs may be the better option. Even then, measure resolver behavior instead of assuming every cache refreshes at the exact expiry instant.

## Verify records, then verify delivery

A deployment check should reject ambiguous records before mail is sent. This Python example validates a captured manifest; it does not pretend that string checks replace live DNS queries or standards-compliant authentication tests.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class TxtRecord:
    owner: str
    ttl: int
    value: str

def validate(records: list[TxtRecord]) -> None:
    if any(record.ttl <= 0 for record in records):
        raise ValueError("TTL values must be positive")
    dmarc = [r for r in records if r.owner.startswith("_dmarc.")]
    if len(dmarc) != 1 or not dmarc[0].value.lower().startswith("v=dmarc1;"):
        raise ValueError("publish exactly one valid DMARC policy record")
    dkim = [r for r in records if "._domainkey." in r.owner]
    if not dkim or any("p=" not in r.value for r in dkim):
        raise ValueError("each DKIM selector needs a public-key field")
```

Record the intended owner, old and new value hashes, old and reduced TTLs, reduction time, earliest safe change time, and external observations. Query authoritative servers and several independent recursive paths; checking only authority skips the caches the plan is intended to control. Then send controlled messages through each legitimate path and retain receiver authentication results. DMARC aggregate reports can confirm broader behavior later, but they are not real-time telemetry.

Set stop conditions. If independent resolvers still return the old value after the calculated window, do not tighten policy. If a new DKIM selector is absent on any tested path, continue signing with the old selector. If SPF testing exposes an unintended sender or approaches the RFC query limit, correct the record before moving traffic. If authentication passes but delivery falls, investigate reputation, content, throttling, and receiver policy rather than repeatedly editing DNS.

Stop means stop.

## Roll out without creating a midnight dependency

Lower the TTL and wait one complete old-TTL interval before beginning the authentication change. Publish additions before activation, observe DNS and message-level results, keep rollback data intact, and schedule removals after the overlap window. Restoring the normal TTL is the final change.

For an emergency during the wait, favor protocol overlap over deletion: keep the existing DKIM key while adding a selector, add a legitimate SPF path before removing another, or pause a DMARC enforcement increase. Rollback should be a precomputed record set with known semantics, not an improvised edit made while receivers disagree.

Choose planned reduction for scheduled clinical-mail changes when overlap is available and authoritative-query reduction matters. Choose permanently brief caching only when unscheduled movement is a documented requirement backed by capacity tests and monitoring. **Judge either policy by externally observed answers and authentication outcomes.** A small TTL printed in a zone is configuration, not evidence.

## Sources

References:

- https://datatracker.ietf.org/doc/html/rfc1034
- https://datatracker.ietf.org/doc/html/rfc2308
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7489
