# User Metadata API: Updating Student Profiles Without Expanding the Auth Boundary

Short answer: a user profile metadata update API should keep only credentials, login identifiers, verification state, recovery factors, and session-control data inside the authentication boundary. Put a student's display name, grade preference, avatar reference, biography, locale, and learning preferences behind the profile API, even when one screen edits several of them. A profile update should not normally invalidate a study session; a primary-email or password change should require stronger proof and should trigger deliberate session handling. The useful design test is not whether a field appears on an account page. It is whether changing that field can change who may authenticate, recover the account, or retain access.

This boundary matters in an education product because friction has a real cost: forcing a learner to re-enter a password whenever they fix a display name can interrupt a lesson, while silently accepting a recovery-email change under an old session can hand the account to somebody else. Those two writes may arrive from the same form, but they do not deserve the same authorization path. Treating them alike produces either needless challenges or a weak trust core.

## What should a user profile metadata update API own?

Start with consequences, not table ownership. Ask a blunt question for every mutable field: **if an attacker changes this value, can the attacker sign in, recover access, defeat a future check, or keep another user out?** If yes, the field belongs in the authentication trust core or must be changed through a command owned by that core. If no, it usually belongs in the profile domain.

That gives an edtech team a workable split. A primary login email, password verifier, email-verification state, recovery factor, failed-login counter, and session revocation state affect account control. A public display name, avatar object key, preferred locale, course-view density, accessibility preference, and student biography do not. Roles and classroom membership are a third category: they are authorization data, not harmless profile decoration, and a general profile endpoint must never accept them.

The dangerous field is the one whose label sounds ordinary. An alternate email can look like contact metadata until a password-reset flow reads it. A school identifier can look like a profile attribute until an enrollment process treats it as proof of membership. The API boundary therefore follows use, not naming. Document every reader of a field before classifying it. If a later feature begins using `contact_email` for recovery, that is a security-boundary change requiring review and migration, not a quiet reuse of an existing column.

Keep the identifiers distinct as well. An immutable internal user ID should connect authentication, profile, enrollment, and audit records. Email is mutable and meaningful to people; using it as the cross-service key turns an email change into a distributed rename and makes partial failure much harder to reason about.

Keep it boring.

## Derive the API from risk, not from the settings screen

A single settings page can issue separate commands. The ordinary profile write can accept a narrow allowlist and the stable subject identifier derived from the authenticated session. It must ignore neither unknown keys nor privileged keys; it should reject them. The client must not choose the target user ID independently of the session unless an explicitly authorized administrative workflow exists.

Sensitive changes take a different path. OWASP recommends reauthentication for sensitive account changes and after risk events such as password resets, account recovery, or suspicious activity. It also recommends invalidating sessions and rotating tokens after reauthentication. That does not mean challenging on every write. It means the server classifies the command, checks how recently and by what method the user proved control, then applies the session policy associated with that command.

The difference is operationally valuable. A learner can correct a misspelled display name without losing a timed exercise. A request to replace the primary email can require recent authentication, verify the new address, notify the old address, and apply the chosen session-revocation rule. A password change can require the current password for an ordinary authenticated flow, while account recovery follows its own guarded process.

Here is a deliberately small policy layer. It does not implement credential storage or email verification; those belong behind the commands it returns. Its job is to prevent a generic metadata patch from acquiring security meaning by accident.

```python
from dataclasses import dataclass
from typing import Any, Mapping

PROFILE_FIELDS = {
    "display_name",
    "avatar_key",
    "locale",
    "learning_preferences",
}
AUTH_FIELDS = {"primary_email", "password"}
FORBIDDEN_FIELDS = {"role", "classroom_ids", "email_verified"}


@dataclass(frozen=True)
class UpdatePlan:
    profile_patch: dict[str, Any]
    auth_commands: tuple[tuple[str, Any], ...]
    requires_recent_authentication: bool


def plan_account_update(payload: Mapping[str, Any]) -> UpdatePlan:
    supplied = set(payload)
    unknown = supplied - PROFILE_FIELDS - AUTH_FIELDS - FORBIDDEN_FIELDS
    forbidden = supplied & FORBIDDEN_FIELDS

    if unknown:
        raise ValueError(f"unknown fields: {sorted(unknown)}")
    if forbidden:
        raise PermissionError(f"fields require another authority: {sorted(forbidden)}")

    profile_patch = {key: payload[key] for key in PROFILE_FIELDS & supplied}
    auth_commands = tuple(
        (f"change_{key}", payload[key]) for key in sorted(AUTH_FIELDS & supplied)
    )

    return UpdatePlan(
        profile_patch=profile_patch,
        auth_commands=auth_commands,
        requires_recent_authentication=bool(auth_commands),
    )
```

This code is intentionally strict. Mass-assignment defenses fail when a new database column becomes writable merely because serialization code discovered it. An allowlist makes adding a field a conscious API decision, while the separate `auth_commands` collection makes it impossible to mistake a credential change for a profile patch. Validation still belongs at each owning service, because an edge-layer check is useful but cannot be the final authority.

The trade-off is explicit: splitting commands adds policy code, audit correlation, and more states for the client to display. This approach is a poor fit for a tiny, single-process application whose fields have one owner and whose authentication data never leaves the same transactional store; a strict modular boundary inside that process may provide the same control with less distributed failure. It becomes worthwhile when different teams or stores own profile, authentication, and authorization data, or when sensitive writes need different evidence and session consequences. Separate services are not the goal. Separate authority is.

## The session decision is a matrix, not a boolean

“Require login” is too coarse. Session security versus friction has at least three decisions: whether an existing session may request the operation, whether fresh proof is required, and what happens to sessions after success. Make those choices per command and record the reason.

| Change | Owning boundary | Proof before write | Session consequence | Principal failure mode |
|---|---|---|---|---|
| Display name or locale | Profile | Valid session | Keep current sessions | Mass assignment into privileged fields |
| Primary email | Authentication | Recent reauthentication plus verification of the new address | Apply an explicit revocation policy | Account takeover through a stolen old session |
| Password | Authentication | Current credential or guarded recovery flow | Rotate or revoke affected sessions | Persistent access after credential compromise |
| Role or classroom membership | Authorization or enrollment | Privileged, purpose-specific command | Re-evaluate authorization | Self-escalation through a profile patch |

Do not infer “recent” from the browser's memory or from the time the settings page loaded. The server needs trustworthy authentication context, and it should fail closed if that context is absent. OWASP also warns that authentication responses can leak whether an account exists through wording, status behavior, or timing; the profile service should not recreate that leak by exposing different public behavior for unknown email addresses.

There is a second trap: success is not complete merely because two rows were updated. If an email change touches authentication state, a verification message, an audit event, and a session store, partial completion must be visible and recoverable. Define one authoritative state transition, make retries idempotent, and publish downstream work from durable state. Do not let the profile database and authentication database race to become the source of truth.

The concrete failure chain is easy to miss during a happy-path review. Imagine that the authentication record accepts the new email, the verification message is queued, and the profile write then times out. A blind retry through the generic endpoint may overwrite the authoritative value with the old profile copy; a second message may carry a different operation; meanwhile an existing session still presents the old display data and support cannot tell whether the change completed. The remedy is not a broader transaction claim that the infrastructure cannot honor. Model a pending transition under one owner, attach an idempotency key to the command, make every consumer accept the immutable subject ID, and expose the pending state to the settings screen. Only the owning boundary declares completion. This costs more state-machine code, but it turns an ambiguous partial write into a transition that can be inspected, retried, and reconciled.

Partial success is failure.

Short failure windows still count. Log a stable operation ID, actor subject, target subject, field category rather than sensitive values, authentication age or assurance decision, outcome, and revocation result. Metrics should distinguish rejected privileged fields, failed recent-authentication checks, verification expiry, and incomplete downstream processing. Never put passwords, password verifiers, recovery tokens, or full session tokens into logs.

## Test the boundary where teams usually test only validation

Unit tests for string length are necessary but uninteresting here. The high-value tests try to cross ownership boundaries: submit `role` beside `display_name`, change the target identifier, replay a sensitive command, retry after a timeout, and present a session created before a password reset. Test that unknown fields fail rather than disappear silently. Test that a successful harmless edit does not revoke a learner's active lesson session.

Then test failure order. What does the user see if the new address is verified but session revocation is delayed? Can the same operation be retried without sending conflicting events? Does an old email remain usable during a pending change, and is that behavior stated in the state machine rather than emerging from callback timing? These are consistency questions. A polished settings screen cannot answer them.

Deployment should begin with observation. Classify existing writes by field, identify every consumer that treats a field as authentication or authorization input, and reject privileged keys in the generic path before moving storage. Shadow the new policy against production-shaped traffic without applying duplicate writes. Once discrepancies are understood, route low-risk profile fields first and sensitive commands last.

Keep rollback asymmetric: ordinary profile writes may return to the old storage path, but a completed credential or identifier change must not be reversed by copying stale profile data back over authoritative authentication state. Backfills should use immutable subject IDs and explicit versions. Counts alone are weak evidence; reconcile hashes or version markers per subject, then sample audit trails for transition order.

## Roll out a smaller trust core

A compact migration can happen in four steps: inventory field consumers, enforce an allowlist on the existing endpoint, introduce purpose-specific commands for account-control changes, and finally move harmless metadata behind the profile service. At each step, preserve one stable subject ID and one authoritative owner per field.

The decision rule remains simple: **authentication owns data whose mutation changes proof, recovery, or session control; profile owns presentation and preference data; authorization owns permissions.** The screen may combine those concepts for usability. The backend should not. That separation lets students make routine corrections with little friction while reserving reauthentication, verification, auditing, and session rotation for the few writes that can actually transfer control of an account.

## Sources

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
