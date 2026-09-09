# Account Deletion Abuse Controls: Device Risk Without Sacrificing Login or Refresh

Short answer: keep ordinary game login fast, rotate refresh credentials on every use, and put account deletion behind a separate, recent proof of identity whose strength rises with device risk; once deletion commits, revoke the account's entire session family in the same authoritative transaction.

That is the architecture decision. A delete request is not merely another authenticated write. In an online game it is both a privacy operation and an attractive abuse primitive: credential-stuffing bots can turn one weak session into irreversible damage, while an attacker holding a copied refresh credential may race the legitimate player's phone, console, or launcher. The security boundary therefore belongs around destructive intent and session lineage, not around every low-risk login.

## What must remain true at the deletion boundary

The system needs four invariants. First, a bearer access credential proves possession, not that the human recently approved deletion. Second, every refresh credential belongs to a server-side family that can be invalidated as a unit. Third, an accepted deletion changes account state and revokes that family atomically, or through a durable state transition that denies access before asynchronous erasure begins. Fourth, retries are idempotent: the same authenticated deletion intent cannot create contradictory account states.

The distinction between access denial and data erasure matters. GDPR Article 17 establishes a right to erasure and also names exceptions; it doesn't prescribe a particular token design or require every byte to disappear inside the request that accepted the user's intent. Treat the privacy workflow as a state machine: `active -> deletion_pending -> access_revoked -> erased`, with retention exceptions reviewed separately. Authentication state should become unusable at the access-revoked boundary, before slower media-object deletion, backups, analytics cleanup, or legal review finishes.

Fail closed.

The failure boundaries are easier to reason about when named. A copied access credential can attempt deletion without a fresh proof. A stolen refresh credential can be replayed after rotation. Two devices can submit refresh and delete operations concurrently. A bot can spray reauthentication attempts to create player lockouts. A worker can process the same erasure event twice. None of those should resurrect access, extend a compromised session family, or turn a transient retry into a second destructive action.

OWASP recommends reauthentication after high-risk events and risk-based authentication when suspicious activity is detected. That supports step-up at deletion, but it does not justify forcing friction into every login. Device risk is an input to the step-up policy — unfamiliar device binding, implausible velocity, recent recovery, or repeated failed proofs may raise assurance requirements — and it must never become the sole authorization decision. I'm not sure any fixed risk threshold stays useful across game genres; replaying labeled decisions and monitoring false challenges is what would resolve that for a particular population.

## How should game account security balance fast login, session refresh, and device risk?

Separate the three clocks. Login establishes a session. Access credentials expire on a short operational horizon chosen from the game's latency and outage tolerance. Refresh credentials last longer but rotate, preserving continuity without granting an unbounded replay window. A destructive action has its own recent-authentication horizon, usually much shorter and backed by a proof appropriate to the account's enrolled factors.

Device signals decide how much proof to request, not whether the account owner deserves privacy rights. A familiar device with a recent strong authentication may only need confirmation; an unrecognized device following password recovery may need a phishing-resistant factor or another enrolled recovery path. Rate limits should bind attempts across account, device, network, and broader abuse clusters, because an account-only counter lets a distributed bot keep trying while a network-only counter punishes shared networks. Exact thresholds are deployment policy, not universal security facts.

Measure both errors.

This split preserves the common path. Players aren't asked to solve the deletion threat during every match reconnect, and security engineers still get a hard boundary where recent proof, replay detection, authorization, audit recording, and account-state change meet. The catch is operational: it requires authoritative session-family state and coordinated writes. It is not suitable when the service cannot make deletion state and revocation agree; in that environment, disable self-service deletion until the control plane can fail closed, while providing a reviewed privacy-request path.

## Compare the control shapes before choosing one

| Control shape | Login latency | Refresh replay response | Deletion assurance | Main failure mode | Appropriate use |
|---|---:|---|---|---|---|
| Stateless access and refresh credentials | Low | No family-wide server decision | Depends on access credential alone | A copied long-lived credential remains useful until expiry | Low-impact sessions with no destructive account action |
| Stateful session family with rotating refresh credentials | Low on ordinary access | Reuse can revoke the family | Still needs recent proof | Concurrent refresh needs strict single-use handling | Most games with multiple devices and account recovery |
| Reauthenticate every login and sensitive action | Higher and frequent | Limits some stolen-session value | Strong when the proof is independent | Bots can amplify prompts and player lockouts | Narrow, high-assurance administrative access |
| Stateful family plus risk-based step-up for deletion | Low on the common path | Reuse revokes related sessions | Recent proof is explicit | Risk policy drift or unavailable enrolled factors | Consumer games that support recovery and multiple devices |

The last shape is the decision here, but it isn't automatically best. It adds a session store, lineage queries, risk-policy operations, and a recovery design. Teams also need to define consistency, durability, and backup behavior for revocation records. A region-local cache is useful for reads, yet it must not be the authority for accepting deletion or rotating a refresh credential; stale acceptance is the dangerous direction. If the authoritative store is unavailable, an existing access credential may retain only the limited behavior already granted by policy, while refresh and deletion wait rather than guessing.

Cost follows state and contention more than cryptography. Model writes per refresh, retained family metadata, cross-region coordination, audit retention, abuse-scoring features, and deletion fan-out. Don't optimize away the one record that lets incident response revoke the attacker's entire lineage.

Caches can lie.

## Put refresh and deletion on one critical state machine

The example below is deliberately an application boundary rather than a framework handler. The transaction object represents an authoritative, serializable decision point; adapters can map it to a database transaction, but they must preserve compare-and-set behavior. `401` means the presented authentication is invalid, `403` means stronger or more recent proof is required, `409` means a concurrent state transition won, and `202` means access is revoked while asynchronous erasure continues.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum


class AccountState(str, Enum):
    ACTIVE = "active"
    DELETION_PENDING = "deletion_pending"


@dataclass(frozen=True)
class DeleteCommand:
    account_id: str
    session_family_id: str
    idempotency_key: str
    proof_id: str
    expected_version: int


def request_account_deletion(command, store, proofs, erasure_queue):
    now = datetime.now(timezone.utc)

    with store.serializable_transaction() as tx:
        prior = tx.find_result(command.account_id, command.idempotency_key)
        if prior is not None:
            return prior

        account = tx.lock_account(command.account_id)
        family = tx.lock_session_family(command.session_family_id)

        if account is None or family is None or family.account_id != account.id:
            return tx.record_result(command, status=401)

        if account.state == AccountState.DELETION_PENDING:
            return tx.record_result(command, status=202)

        proof = proofs.verify_recent(command.proof_id, account.id, now)
        if not proof.valid or not proof.satisfies(account.required_delete_assurance):
            return tx.record_result(command, status=403)

        if account.version != command.expected_version:
            return tx.record_result(command, status=409)

        tx.update_account(
            account.id,
            state=AccountState.DELETION_PENDING,
            next_version=account.version + 1,
        )
        tx.revoke_all_session_families(account.id, revoked_at=now)
        tx.append_audit_event(
            account.id,
            event_type="account_deletion_accepted",
            occurred_at=now,
        )
        tx.enqueue_outbox_event(
            topic="account_erasure_requested",
            key=account.id,
        )
        result = tx.record_result(command, status=202)

    erasure_queue.publish_pending_outbox()
    return result
```

There is an important ordering detail: the durable outbox entry is written with revocation and account state, while publication happens afterward. If delivery is repeated, downstream deletion workers deduplicate on the account and transition version. The public response reveals neither whether a guessed account exists nor which risk signal fired; detailed reasons belong in access-controlled audit data, with minimization and retention limits of their own.

Retries are normal.

Refresh uses the same discipline. Lock the family, hash and compare the presented credential, reject a revoked family, consume the current credential exactly once, and issue its successor inside one transaction. If a consumed credential appears again, treat that as replay and revoke the family. Consider the concrete race: a phone presents refresh credential R7 while an attacker presents the copied R7 from another device. The transaction that commits first consumes R7 and creates R8; the second transaction cannot also create a valid branch, so it records reuse and revokes the family. It does not matter which caller won. Both must authenticate again, because preserving the winner would let timing decide whether the attacker retained access. OAuth 2.0 Security Best Current Practice describes refresh-token rotation and sender-constrained refresh tokens as methods for detecting replay in public clients; a game backend should select based on client capability and threat model rather than pretending a launcher can safely keep a permanent secret.

Test the races, not only the happy path. Run refresh-versus-refresh, refresh-versus-delete, duplicate-delete, delayed-outbox, restored-backup, and region-failover cases. Assert the invariant after each schedule: once deletion is accepted, no session family for that account can mint new access. Observability should count step-up challenges, proof failures, refresh reuse, family revocations, deletion latency by state, duplicate events, and denied post-deletion refresh attempts, but logs must not contain raw credentials or unnecessary device fingerprints.

## Record the rejected option and its valid boundary

The rejected design is a completely stateless refresh credential plus deletion authorized by any valid access credential. Its appeal is real: fewer writes, fewer coordinated failure modes, and straightforward horizontal scaling. It is still a reasonable fit for a low-impact anonymous profile where the only state is disposable and the credential lifetime is short.

It is the wrong fit for a durable game account containing purchases, social identity, user-generated media, or an erasure obligation. There is no authoritative family record to revoke across devices, and a destructive operation cannot distinguish an old stolen bearer credential from a recent deliberate approval. Shortening every credential until that risk feels acceptable shifts the cost onto honest reconnects and still does not create proof of destructive intent.

The alternative rejected at the other extreme is mandatory step-up at every login. Stick with it for a small administrative console where each session controls high-impact operations and users have managed authenticators. For a consumer game, repeated prompts enlarge the bot-driven denial-of-service surface and train players to approve challenges mechanically. Risk-based step-up at the destructive boundary has more moving parts, but those parts correspond to the actual decision being protected.

Ship the state machine behind staged policy, shadow-score device risk before it can challenge users, and rehearse account recovery before enabling self-service deletion. The decision is acceptable only when session revocation is authoritative, deletion retries are idempotent, and a player who loses every device still has a documented recovery route that does not let support bypass the same assurance model.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc9700.html
- https://eur-lex.europa.eu/eli/reg/2016/679/art_17/oj
- https://pages.nist.gov/800-63-4/sp800-63b.html
