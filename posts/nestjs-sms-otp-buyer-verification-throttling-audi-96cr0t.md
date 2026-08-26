# NestJS SMS OTP Buyer Verification (Throttling, Audit Logs, and Recovery Codes)

**Short answer:** Build marketplace buyer verification as a state machine around NestJS, not as a controller that sends a six-digit string: throttle issuance and guesses independently, store keyed OTP digests, consume each challenge or recovery code atomically, and retain a minimal audit trail that proves the decision without retaining the secret.

For an education marketplace, the least complex defensible design has three ports behind the NestJS service: a transactional challenge store, an SMS transport, and an append-only audit sink. The phone-possession result and the seller's new-order notification are separate facts. Link them with a correlation ID, but don't let a delivery receipt masquerade as buyer verification.

Keep those claims separate.

## What does SMS OTP evidence cost to keep?

Start with the bill because it exposes a design mistake early. The variable transport term is `SMS dispatches x current per-message charge`; the evidence term is `audit events x average event size x retention period`; operational cost includes support investigations and reconciliation. Measure all three in the deployment region instead of copying a rate card into architecture. For a concrete capacity calculation, 100,000 accepted challenge requests with 1.25 dispatches per challenge produce 125,000 billable dispatches, while five retained audit events per challenge produce 500,000 event rows. Those are workload inputs, not a claim about any provider or marketplace.

Dispatch amplification usually deserves the first investigation: retries, impatient repeat requests, and duplicated workers can turn one buyer intent into several messages. An idempotency key on challenge creation, a cooldown on another request, and an outbox record for dispatch move that term. Audit retention is different. Events are small, but the liability grows with the fields copied into them and with how long raw identifiers remain searchable. Keep policy version, buyer reference, challenge reference, correlation ID, event type, event time, template version, and a provider message reference when one exists. Don't keep the OTP, a recovery code, the submitted guess, the rendered message, or a full phone number in authentication logs.

The deliberate loss is content-level reconstruction. After the approved retention window, delete or irreversibly de-identify destination references and old transport metadata according to the marketplace's policy. A later dispute may then prove that a versioned verification decision occurred without recovering the original message or phone number. That trade is uncomfortable — it should be. Retaining everything makes investigations easy until the evidence store itself becomes the incident.

## How should a NestJS SMS OTP backend throttle marketplace buyer verification?

Use two limiters because they defend different resources. Issuance limits protect a recipient from message flooding and protect the messaging budget; attempt limits protect a low-entropy code from repeated guesses. Key issuance by both the authenticated buyer and a privacy-preserving destination reference. A buyer-only key is weak against fresh accounts, while a destination-only key lets an attacker consume someone else's quota. Network and device signals can add context, but shared networks make blunt IP blocking a poor sole control.

Guess accounting belongs to the challenge row. A repeat request creates a new challenge and supersedes the old one; it must not replenish guesses on an existing challenge. Return one generic outcome such as `OTP_INVALID_OR_EXPIRED` for a wrong, expired, locked, superseded, or already-consumed challenge, so the API doesn't reveal useful state to a guesser. A throttled issuance response can expose a bounded `retryAfterSeconds` because the caller needs to render the cooldown, but the server remains authoritative.

Here is the small state transition I would specify before wiring controllers or an SMS adapter. The Python is an executable model of the transaction that the NestJS store must provide; the important contract is compare-and-set, not the framework syntax.

```python
from dataclasses import dataclass, replace
from enum import Enum


class Status(str, Enum):
    PENDING = "pending"
    VERIFIED = "verified"
    LOCKED = "locked"
    EXPIRED = "expired"


@dataclass(frozen=True)
class Challenge:
    challenge_id: str
    buyer_id: str
    otp_digest: bytes
    expires_at_ms: int
    attempts_left: int
    version: int
    status: Status


def decide_verification(
    challenge: Challenge, *, now_ms: int, digest_matches: bool
) -> tuple[Challenge, str]:
    usable = (
        challenge.status is Status.PENDING
        and challenge.expires_at_ms > now_ms
        and challenge.attempts_left > 0
    )
    if not usable:
        return challenge, "OTP_INVALID_OR_EXPIRED"

    if digest_matches:
        return (
            replace(
                challenge,
                status=Status.VERIFIED,
                version=challenge.version + 1,
            ),
            "OTP_VERIFIED",
        )

    remaining = challenge.attempts_left - 1
    return (
        replace(
            challenge,
            attempts_left=remaining,
            status=Status.LOCKED if remaining == 0 else Status.PENDING,
            version=challenge.version + 1,
        ),
        "OTP_INVALID_OR_EXPIRED",
    )
```

The repository reads version `n` and updates only if the stored version is still `n`. Two simultaneous correct submissions may both calculate `OTP_VERIFIED`, yet only one update can commit version `n + 1`; the loser gets the generic rejection. Generate the OTP with a cryptographically secure random source, bind its keyed digest to the challenge ID, and compare digests without data-dependent early exit. NestJS should validate input and map the domain result to HTTP, but the database transaction owns single consumption.

One commit wins.

Concrete policy values belong in versioned configuration. Five minutes, three issuance requests per 60 seconds, and five guesses can make a useful test fixture, but they aren't universal recommendations. Marketplace risk, destination abuse, delivery delay, and support recovery all affect the right values. I'm not sure a single expiry or attempt count is defensible across jurisdictions and buyer populations; a documented threat model plus measured delivery latency would resolve that question.

## Integrate atomic state with the audit boundary

An audit event should answer who initiated a transition, which policy was applied, what state change was accepted, and when it committed. It shouldn't become a second credential database. Record `otp.requested`, `otp.dispatched`, `otp.rejected`, `otp.verified`, `recovery.consumed`, and administrative recovery changes with stable identifiers and reason classes. Keep the actor type as well: buyer, support operator, scheduled expiry process, or authorized agent. Never record plaintext secrets or arbitrary tool input. Ordering matters. Create the challenge and an outbox item in one transaction, then let a worker dispatch the message once for that outbox item and append the provider reference. Verification commits the state transition and its audit event together when compliance evidence is mandatory. The catch is latency and availability: coupling evidence to the critical transaction adds storage work and makes evidence availability part of authentication availability. For a marketplace action that legally depends on proof, that is often the honest boundary. For a low-risk preference check, an asynchronously retried audit append may be acceptable, but it can leave a temporary evidence gap and should be labeled as such in the control design. Templates deserve a smaller role than they often get. A server-owned Mustache template may receive an allowlisted object such as a code and expiry text; the buyer cannot supply template source, recipient, or template identifier. Mustache's documented variable and escaping behavior makes the rendering rules reviewable, but rendering must never decide authorization, expiry, or state. Store the approved template version in the dispatch event, not the rendered body. If an LLM agent can begin verification, expose a narrow tool like `request_buyer_verification` with a schema containing the buyer reference and order reference. The tool-use boundary described in Anthropic's guide is useful here because the model asks to invoke a defined operation; ordinary backend authorization, destination ownership, throttling, and evidence policy still decide whether it runs. A generic `send_sms` tool with arbitrary recipient and body fields creates authority the workflow doesn't need.

## Govern recovery data before it becomes permanent

Recovery codes aren't long-lived OTPs. Generate a set only after an appropriate enrollment decision, show plaintext once, store an individually salted or keyed digest for each entry, and give each record its own `consumed_at` field. Redemption is another conditional update: one unused digest becomes consumed and one audit event commits. Concurrent reuse loses. Don't silently issue a fresh set when a buyer loses a phone or exhausts SMS guesses. Phone change, account recovery, and OTP lockout are different risk events. Define which authenticated session or reviewed support process may rotate the set, invalidate the old set as one operation, and notify through an independently established channel where policy requires it. This path may be unsuitable for a marketplace that cannot protect one-time display, support secure storage guidance, or operate a reviewed reset process; in that case, use a separately governed support recovery flow rather than pretending printable codes are harmless. There is also a retention asymmetry. The service needs active recovery-code digests until use, rotation, or expiry, but the audit system usually needs only the code-set reference, event, actor, policy version, and time. Keeping a digest forever after consumption adds little evidentiary value. Removing it makes later forensic rechecking impossible, so security and compliance owners must choose and document that boundary rather than allowing the database default to choose it accidentally.

Recovery is its own control plane.

## Rollout gates for buyer verification

Transport integration is the adapter around the harder work. Require a stable internal dispatch result, an idempotent send operation, authenticated and deduplicated delivery callbacks if callbacks affect workflow, and timing fields that support reconciliation. An SDK can reduce authentication and typing work but expands the dependency surface; direct HTTP can make replacement clearer but leaves retry rules and maintenance with the team. A self-hosted gateway increases operational control, yet it is not suitable when the team cannot own carrier relationships, abuse response, deliverability, and on-call work. No option removes the state machine.

Test concurrent correct submissions, a wrong guess at the last permitted attempt, repeat requests on both sides of the cooldown boundary, verification at expiry, duplicated queue delivery, repeated callbacks, recovery-code reuse, template rotation during an active challenge, and audit-store unavailability. Check the invariant, not merely the status code: no sequence may produce two successful consumptions, replenish guesses, or write a secret to logs. Include `correlationId` in application and worker logs, then verify through automated log inspection that OTPs, recovery codes, full message bodies, and raw authorization headers are absent.

Deployment should expose counts and latency distributions for accepted issuance, suppressed issuance, dispatch, verification outcomes, recovery use, and audit append. Alert thresholds need observed baselines; invented universal percentages won't help. Rehearse digest-key rotation, retention deletion, and a buyer dispute using only approved evidence. If an investigator needs an improvised join across production tables to explain one decision, the evidence model isn't ready.

Choose the SMS transport only after that rehearsal. The final decision should turn on callback authenticity, idempotency support, regional delivery requirements, evidence export, team ownership, and current total cost under the measured dispatch pattern. Stick with the existing transport when it satisfies those controls and replacement would add migration risk; change it when a documented requirement cannot be met. The transport is replaceable. The meaning of `verified` cannot be.

## References

- https://mustache.github.io/mustache.5.html
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview

## Further reading

- https://mustache.github.io/mustache.5.html
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
