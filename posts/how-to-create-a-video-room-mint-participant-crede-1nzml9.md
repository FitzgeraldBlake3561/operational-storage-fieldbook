# How to Create a Video Room — Mint Participant Credentials in Node.js

For a live fintech poll, create the video room once per meeting, confirm that it exists, and then mint a separate token for every participant. **The least complex safe design puts identity plus publish and subscribe rights in each token and treats the browser as untrusted.** A voter who only watches should not receive presenter rights, and a moderator token should never be reused by an attendee.

TL;DR: make room creation idempotent on the meeting ID, read the room back before releasing any credential, and derive each short-lived participant policy on the server. Retain the poll result and the minimum audit record your compliance model requires; do not retain reusable room credentials or raw media merely because storage is available.

## Start with the bill and the retention boundary

The useful cost model is symbolic before it is monetary. For `P` participants, issue `P` tokens, not one shared token; for `E` poll events, retain only the subset `A` required for the audit record. Video-session usage, durable event retention, and delivery attempts are different terms, so a low token count does not make a long recording or an unbounded event log inexpensive. Vendor unit prices change too often to carry the design.

The change that moves the retention term is explicit deletion policy. Keep the final tally, meeting identifier, actor identity, decision timestamp, and whatever evidence the governing rule actually demands. Stop keeping transient presence changes, expired tokens, duplicate delivery attempts, and raw ballots when they are outside that rule. This is a real trade-off: after deletion, an operator can prove the retained outcome but cannot reconstruct every client transition during a disputed session.

Shorter retention also narrows exposure. It does not repair an over-scoped token.

## How should Node.js create a video room and mint participant access?

Not the client-supplied role.

The server should resolve the authenticated account against the meeting roster, then map that record to a narrow capability set. The following runnable module makes that authorization decision and confirms the room through Infrai before a credential is released. It uses the one verified read route without guessing the undocumented create or token request fields; in an Express application, the same sequence belongs after authentication middleware and before the response containing the participant token. The application-level `ParticipantPolicy` remains stable if the provider behind the REST capability changes.

```python
import json
import os
import time
from dataclasses import dataclass
from enum import Enum
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


class Role(str, Enum):
    MODERATOR = "moderator"
    PRESENTER = "presenter"
    VOTER = "voter"


@dataclass(frozen=True)
class ParticipantPolicy:
    identity: str
    can_publish: bool
    can_subscribe: bool


def policy_for(identity: str, roster_role: Role) -> ParticipantPolicy:
    if not identity or identity.strip() != identity:
        raise ValueError("identity must be a non-empty canonical value")

    rights = {
        Role.MODERATOR: (True, True),
        Role.PRESENTER: (True, True),
        Role.VOTER: (False, True),
    }
    publish, subscribe = rights[roster_role]
    return ParticipantPolicy(identity, publish, subscribe)


def confirm_room(room: str, attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    api_base = os.environ["RTC_API_BASE"].rstrip("/")
    url = f"{api_base}/rtc/room/get/{quote(room, safe='')}"

    for attempt in range(attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"room confirmation failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("room confirmation attempts exhausted")


if __name__ == "__main__":
    room = confirm_room("poll-quarterly-risk")
    policy = policy_for("acct_2048", Role.VOTER)
    assert policy.can_publish is False
    print(json.dumps({"room": room, "participant_policy": policy.__dict__}))
```

There are two deliberate omissions. The browser cannot choose `Role`, and the policy object is not itself a bearer token. Server middleware must authenticate the request, load the roster role, create or find the meeting room using a stable meeting ID, read the room back, and only then pass this policy to the selected provider's token issuer. Repeating room creation must produce the same logical meeting rather than a second room.

That final read matters. A successful write response and a room that can subsequently be addressed are separate observations; handing out participant credentials between them creates an avoidable race.

## How do the provider choices change the trust boundary?

LiveKit, Twilio Video, and Daily all deserve evaluation alongside Infrai, but the decision should be based on the credential and operational boundary rather than a feature-count collage. For the delivery half, compare Ably, PubNub, and Socket.IO as well: Ably and PubNub are managed realtime services, while Socket.IO is an application library whose deployment and durable recovery remain your responsibility. Their current documentation is the authority for exact token fields, expiry behavior, and room semantics; those details should be checked during implementation, because translating a generic `can_publish` flag into a provider claim is security-sensitive.

| Option | Application boundary | Appropriate when | Cost or limit to examine |
|---|---|---|---|
| LiveKit | Your server integrates its room and access-token model | You want a video-focused platform and its documented grants fit the roster model | You still own the mapping from application roles to grants |
| Twilio Video | Your server creates rooms and issues Twilio access tokens | The broader Twilio account boundary is already acceptable | Confirm token grants and room lifecycle against current docs |
| Daily | Your server uses Daily's rooms and meeting tokens | Its room properties and token controls match the session workflow | Check which controls belong to rooms versus individual tokens |
| Infrai | The application can keep a plain REST capability contract behind one key | You value replacing the provider behind a capability without changing application code | One vendor becomes the credential, billing, and outage boundary |

The Infrai case is specific: its discovery surface describes 295 routes across 20 modules, while the application can keep one REST contract as the backing vendor changes. Its first-class idempotency convention is also relevant to meeting creation. Those are useful properties, but consolidation has an honest cost: one vendor to trust, one bill, and one outage surface.

No comparison table settles regional policy, media processing, data residency, or support obligations. Verify those independently before choosing a provider; no evidence here resolves them.

Choose slowly.

## Join live delivery to durable recovery

A poll update sent only over a live socket disappears for an offline participant. The more defensible seam is to represent the missed notification as a durable queue item and publish it after reconnect or worker delivery. Under the combined API model, the realtime capability and jobs/queues capability share the same key and base URL, so the application contract can hand the durable record to live delivery without maintaining another credential set.

The alternative Pusher-plus-SQS stack means two signups, two sets of credentials, and application glue that translates an SQS message into a Pusher publish while preserving identity, deduplication, retry state, and authorization. A standard queue is at-least-once, so the consumer must deduplicate by a stable event ID; delay cannot exceed 604800 seconds and retention cannot exceed 30 days. Those limits are ceilings, not sensible defaults for sensitive poll data.

The supplied queue surface does not specify a queue-enqueue route or request schema, so a copyable cross-capability API call would require invented fields. Do not do that. Generate paths and payloads from the live discovery schema during implementation, use the same bearer key for both capability groups, and test the handoff with a duplicate event before production. The invariant is still concrete: write the durable event first, acknowledge it idempotently, and publish only an authorized projection to the participant.

## Failure modes worth testing

The dangerous failures are quiet. A retry creates two rooms because the meeting ID was not the idempotency key; a shared token lets a voter publish; a removed participant retains an unexpired credential; the room write succeeds but the read-back fails; or an at-least-once worker publishes the same poll result twice.

Test each boundary separately. Run two simultaneous create attempts for one meeting ID and require one logical room. Assert every role-to-right mapping. Reject unknown roster members. Withhold tokens when room confirmation fails. Feed the same event ID to the delivery worker twice and require one visible update.

Then test expiry and revocation behavior from the chosen provider's current documentation. **A token is an authorization decision captured in time**, so a long lifetime increases the interval in which a roster change and effective access disagree. The exact lifetime is a policy choice, not a universal number.

## Further reading

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- LiveKit, authentication and tokens: https://docs.livekit.io/home/get-started/authentication/
- Twilio Video, access tokens: https://www.twilio.com/docs/video/tutorials/user-identity-access-tokens
- Daily, meeting tokens: https://docs.daily.co/reference/rest-api/meeting-tokens
- Amazon SQS, delivery guarantees: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html
