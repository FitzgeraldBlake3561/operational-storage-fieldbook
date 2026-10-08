# Node.js Marketplace Access Review — API Key Inventory or Application Audit Logs

Short answer: run a leaked-key drill with two evidence sets, because an API key inventory proves which credentials could have opened an account while application audit logs prove which tenant records were actually reached. Treat neither as a substitute for the other. The deciding constraint is auditability: investigators must join a stable credential identifier to actor, tenant, request, authorization decision, and outcome without storing the secret itself.

For a marketplace access review, the practical question is not merely whether a key existed. It is whether that key could cross a seller boundary, whether it did, and what must be revoked or repaired. The architecture decision is to maintain an authoritative credential registry beside an append-oriented application event stream, then test the join during every leaked-key drill.

## Can API key inventory or application audit logs answer an access review?

Start with four invariants. Every issued credential has an opaque, non-secret ID; every authenticated request carries that ID into authorization and logging context; every tenant-scoped decision records both the requested tenant and the resolved tenant; and revocation changes future authorization behavior while preserving prior evidence. A fingerprint can help operators recognize a presented key, but it must not become a reusable authenticator or a casual substitute for a credential ID. OWASP advises assigning secrets to the narrowest practical scope, rotating them, revoking them, and auditing their lifecycle.

The failure boundaries matter more than the happy path. The inventory can say a seller-integration key was active from 09:00 to 10:15 UTC, scoped to tenant `seller_482`, and revoked after disclosure. It cannot establish that any order was read. Conversely, an event saying an order export succeeded is weak evidence if it lacks the credential ID, tenant resolution, authorization result, and request correlation needed to attribute that access.

Inventory is necessary. It is also insufficient.

Attribution is the hard part.

During the drill, answer three questions in order: what authority was exposed, where that authority was exercised, and whether containment invalidated every derived path. The last question includes replicas, caches, queued work, and already-issued sessions if the system can exchange an API key for another token. A key marked `revoked` in one database is not containment evidence until an integration test shows the enforcement path rejects it.

Make the exercise bounded enough to repeat. For example, issue a synthetic seller credential, run a documented 60-minute scenario, revoke it at the recorded midpoint, and preserve the expected requests alongside the observed evidence. The specific duration is a test parameter, not a security guarantee. What matters is that another reviewer can rerun the same sequence and explain every mismatch instead of guessing whether silence means no access or lost telemetry.

## Evidence boundaries and named failure modes

| Evidence source | Question it can answer | Required fields | Failure mode it cannot resolve |
|---|---|---|---|
| Credential inventory | Which authority existed during the exposure window? | credential ID, owner, tenant scope, permissions, issued time, state, rotated/revoked time | Use is invisible when requests never carry the ID forward |
| Application audit stream | What protected action was attempted and decided? | event ID, time, credential ID, actor, requested and resolved tenant, action, resource class, decision, outcome, request ID | Exposure scope is ambiguous when lifecycle history is missing |
| Infrastructure access log | Which network request reached an edge or service? | time, route template, status, request ID, service identity | A `200` does not prove which business records were authorized or returned |
| Queue or worker event | Which deferred action ran after the request? | originating request ID, job ID, tenant, credential ID or delegated actor, outcome | Attribution breaks when job payloads drop the initiating identity |

There are predictable ways this design fails. Secret values leak into logs when middleware records headers. Tenant context is inferred from a user-controlled path but never compared with the credential's server-side scope. Rotation overwrites one inventory row, destroying the timeline. A background job records only a service account, erasing the initiating credential. Clock skew makes a revocation appear to precede a successful request. Retry handling emits multiple events without a stable event or request ID. These are forensic defects even when production traffic still succeeds.

Logs can fail too.

Audit storage has a different threat model from ordinary diagnostics. OWASP's Logging Cheat Sheet recommends excluding access tokens and other primary secrets from logs, sanitizing event data, recording enough context for analysis, and protecting logs from tampering and unauthorized access. Retention must follow the marketplace's legal and operational requirements; no universal duration can be inferred from the existence of an API key. Record the policy and verify it with the oldest query the drill requires.

Retention is a requirement, not a default.

## The critical path in code

The following Python is deliberately framework-neutral even though the service boundary may be implemented in Node.js. It shows the contract that matters: authenticate to an internal credential ID, resolve scope on the server, emit one structured decision event, and never log the presented secret. Production implementations also need transactional or durable delivery guarantees appropriate to their data layer; an in-process logger alone does not provide them.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol
from uuid import uuid4


@dataclass(frozen=True)
class Principal:
    credential_id: str
    actor_id: str
    tenant_id: str
    permissions: frozenset[str]


class AuditSink(Protocol):
    def append(self, event: dict) -> None: ...


def authorize_order_export(
    principal: Principal,
    requested_tenant: str,
    request_id: str,
    audit: AuditSink,
) -> bool:
    resolved_tenant = principal.tenant_id
    allowed = (
        requested_tenant == resolved_tenant
        and "orders:export" in principal.permissions
    )
    audit.append({
        "event_id": str(uuid4()),
        "occurred_at": datetime.now(timezone.utc).isoformat(),
        "request_id": request_id,
        "credential_id": principal.credential_id,
        "actor_id": principal.actor_id,
        "requested_tenant_id": requested_tenant,
        "resolved_tenant_id": resolved_tenant,
        "action": "orders.export",
        "resource_type": "order_collection",
        "decision": "allow" if allowed else "deny",
        "outcome": "authorized" if allowed else "blocked",
    })
    return allowed
```

The code stops at the decision boundary on purpose. A real export should emit or correlate a later completion event, because `authorized` does not mean bytes were delivered. If audit publication and the protected state change are separate writes, process termination can leave one without the other; a transactional outbox can close that gap when both records share a transactional database, while an idempotent consumer prevents retries from inventing extra actions. Where no shared transaction exists, document the residual loss and duplication windows and test them.

Authorization is not delivery.

The drill should include at least four fixtures: an allowed request within the seller tenant, a denied request against another tenant, a queued export that runs after revocation, and a duplicated delivery with the same event ID. Search by credential ID, then reconcile the event count against authoritative business effects. Also verify the negative case: searches for the full leaked secret and recognizable prefixes must return nothing in application, edge, trace, and error stores.

## Comparing implementation choices

| Design | Auditability | Consistency boundary | Operational cost | Appropriate use |
|---|---|---|---|---|
| Separate inventory and structured event stream with stable IDs | Strong when joins and retention are tested | Lifecycle updates and events require explicit correlation | Two schemas, access controls, and recovery procedures | Marketplace actions where tenant attribution and historical credential state both matter |
| One mutable credential table with `last_used_at` | Weak; later use overwrites history | Simple row update | Low storage and query complexity | Coarse operational hints where incident reconstruction is explicitly out of scope |
| Raw gateway logs plus secret fingerprints | Moderate for request arrival, weak for business authorization | Edge observation only | Existing log pipeline may be reused | Network triage, rate analysis, and corroboration |
| Application events without a credential registry | Strong action detail, weak exposure history | Business event boundary | Event governance still required | Actor types whose lifecycle authority is maintained in another authoritative system |

Storage branding does not change these boundaries. GitHub, GitLab, and Bitbucket expose different audit and token-management surfaces, but their documented records are platform evidence, not automatic proof of a marketplace application's tenant-level authorization decision. If one hosts source or automation, its audit data may corroborate who changed a secret or workflow; the application still owns the event that says which seller resource was allowed. Evaluate any platform by exported fields, retention, timestamp semantics, immutability controls, and the ability to join its identity to your internal credential ID.

The same skepticism applies to durability claims. An append-only API is not proof of immutability if administrators can rewrite the backing store without a second control. Object lock, restricted deletion, cryptographic chaining, and independent replication address different threats. Choose controls after naming the attacker and recovery objective, then run a restore query; a bucket full of unreadable events is durable storage but failed evidence.

Test the restore.

The limitation of the two-store design is operational, not cosmetic: teams must govern two retention paths, keep their identifiers compatible, restrict two sets of readers, and rehearse recovery for both. Its trade-off is justified where cross-tenant marketplace access has to be reconstructed after the fact. It is not suitable for a low-risk internal script whose only requirement is immediate revocation and whose actions are already recorded, with the initiating identity, in an authoritative system; there, a separate duplicate event store adds failure modes without adding evidence.

## Rejected option and the case where it still fits

We reject `last_used_at` as the primary audit mechanism for the leaked-key drill. It collapses multiple requests into one timestamp, carries no resource or authorization decision, races under concurrent use, and cannot distinguish an allowed seller export from a denied cross-tenant attempt. Adding `last_ip` merely produces a slightly richer mutable summary. It still cannot reconstruct access.

The option remains valid for a narrow product signal: showing an owner that a credential appears dormant before planned rotation. Label it as advisory, define how delayed writes affect it, and do not let its apparent precision drive incident scope. A mutable usage hint can improve routine hygiene without pretending to be an audit trail.

The acceptance rule is concrete: given one synthetic leaked credential ID and a bounded time window, an investigator can enumerate its historical authority, find every authorization decision across request and worker paths, prove tenant isolation for denied cases, link allowed decisions to business outcomes, and demonstrate rejection after revocation. Any missing join is a failed drill, not a dashboard inconvenience.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc7519
- https://csrc.nist.gov/publications/detail/sp/800-92/final
- https://docs.github.com/en/enterprise-cloud@latest/admin/monitoring-activity-in-your-enterprise/reviewing-audit-logs-for-your-enterprise/about-the-audit-log-for-your-enterprise
- https://docs.gitlab.com/administration/audit_event_streaming/
- https://support.atlassian.com/security-and-access-policies/docs/track-organization-activities-from-the-audit-log/
