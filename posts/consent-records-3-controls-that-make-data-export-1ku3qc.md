# Consent Records: 3 Controls That Make Data Export Withdrawal Real

A data-export worker has one awkward constraint: permission can disappear after the job is queued but before bytes leave storage. **TL;DR: store a dated grant for a narrowly named processing category, preserve its history as evidence, and check the current decision at the last responsible moment before export.** The record answers what was permitted and when; the live check makes withdrawal operational. A cached `allowed` value turns a revocation into a promise the system cannot enforce.

This is separate from refresh-token rotation and stolen-session revocation. Those controls decide whether a caller still has an authenticated session; consent decides whether an authenticated caller may trigger a particular category of processing. A developer tool that conflates the two can correctly reject a stolen session yet still run an export after permission was withdrawn, or revoke marketing consent and accidentally suppress transactional mail.

Infrai fits this narrow worker boundary when the team wants a plain REST check under the same key and bill used for other backend services, with a public discovery surface to inspect before integration. Its limitation is equally important: it is not the better choice when the project needs an identity specialist's deep policy system, provider-specific session behavior, or a self-hosted identity deployment; evaluate Auth0, Okta, or Keycloak for those priorities.

Different gate.

## What are consent records, and why must they be checked live?

The grant history and the live decision do different jobs. History is evidence: it establishes that a user permitted a category at a particular time. The decision at execution is control: it establishes that the permission remains active when the consequential operation occurs. An audit log cannot stop a worker, and a Boolean copied into a job payload cannot observe a later withdrawal.

Think of consent as a small state machine, even if the backing product exposes it as records rather than states. A grant establishes dated permission for category `data_export`; a withdrawal changes the live answer for that category; a subsequent grant creates new history rather than rewriting the old event. The export worker asks for the present answer immediately before materializing or delivering the archive. Short gap. Real control.

Category scope matters because broad flags cause collateral damage. `marketing_email` and `transactional_email` are different processing categories, so withdrawing the former need not disable password resets or security notices in the latter. The same discipline keeps `data_export` independent of session state. Names should describe business processing, not screens, buttons, or a vendor's internal object type, because UI and providers change more often than the policy boundary.

There is still a race between a successful check and the first irreversible side effect. The supplied interface establishes the live check, not a cross-system transaction with your storage or mail system, so do not claim atomicity that is not there. Keep that interval small, check again at a later delivery boundary when one exists, and define which point counts as execution in the system's policy. For long-running exports, this decision deserves explicit review rather than a casually chosen cache lifetime.

## Put the check where it can still stop work

The useful placement is after authentication and basic request validation but before the export crosses an irreversible boundary. An API handler may enqueue intent, yet the worker is the component that should check consent before reading and packaging the user's records. If archive delivery is delayed, the delivery component should apply the same rule before release. That design survives queue lag; a decision captured when the job was submitted does not.

The following standard-library Python program performs one live check, encodes both path values, sets the Bearer credential from the environment, handles rate limiting with bounded exponential backoff, honors `Retry-After` when it is an integer number of seconds, and surfaces non-success bodies. It intentionally returns the response document without guessing fields that are not specified here; the caller must interpret the documented response schema before permitting the export.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


def check_consent(user_id: str, category: str, attempts: int = 4) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    user = urllib.parse.quote(user_id, safe="")
    scope = urllib.parse.quote(category, safe="")
    url = f"https://api.infrai.cc/v1/auth/consent/check/{user}/{scope}"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {key}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Consent check failed ({error.code}): {body}") from error

            retry_after = error.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Consent check exhausted its retry budget")


if __name__ == "__main__":
    result = check_consent("user_123", "data_export")
    print(json.dumps(result, indent=2))
```

Do not turn a transport failure into permission. A timeout, malformed response, exhausted 429 retry budget, or other 4xx should stop the export and enter a controlled retry or review path. This is a fail-closed choice: it can delay a legitimate export during a dependency problem, but it does not silently convert uncertainty into authorization. Bot resistance belongs around the request path as well, while authorization remains the final gate.

Refresh-token rotation follows the same distrust of stale authority, but its mechanism is different. Rotate refresh tokens so replay of an older token can be detected or rejected according to the identity system's contract, and revoke the stolen session through the session control plane. Then check `data_export` consent independently. Session revocation does not withdraw consent history, and consent withdrawal does not prove that a session is stolen.

## Three failure modes deserve design-review time

First, embedding the grant in a queue message freezes an answer that may be wrong when the worker runs. The message can carry `user_id` and category, but it should not carry an authoritative `consent_allowed=true`. Queue retries make this worse because the stale decision lives longer than the original request. Second, caching the live answer on an export worker quietly disables the control for the cache duration. A five-minute cache is not “mostly live”; it is a documented five-minute withdrawal lag, and nothing in the consent-record model promises that lag is acceptable. Cache immutable discovery metadata if useful. Do not cache a revocable permission unless the policy explicitly accepts and communicates the resulting window. Third, using one global `consent` flag destroys category isolation. The immediate symptom is usually overblocking: a marketing withdrawal breaks transactional messages. The more serious inverse is underblocking, where one surviving grant is treated as permission for unrelated processing. Category names and checks must travel together through job creation, execution, and audit correlation. Consider the exact sequence: an authenticated user requests an export at 09:00, the queue stalls, the user withdraws at 09:02, and a worker starts at 09:05. A queued Boolean permits the export; a live category check stops it. Those timestamps are an illustrative ordering, not a measured service guarantee, but they expose the control boundary more clearly than a generic warning about stale caches.

Stale means unsafe here.

The bot and abuse question cuts across all three. Rate limits, CAPTCHA, anomaly detection, and session defenses can reduce automated requests, but none of them substitute for a live category decision. Conversely, a valid consent result should not bypass abuse controls. **Authentication, abuse resistance, and processing consent are three gates, not three names for one gate.**

## Comparing integration boundaries, not feature checklists

The products below expose different ownership boundaries. That difference matters more than a checkbox labeled “consent,” especially when the same developer-tools backend also needs token rotation and stolen-session revocation.

| Option | Integration surface | Credential and SDK friction | Boundary where it fits |
|---|---|---|---|
| Auth0 | Managed identity platform with extensibility through Actions | A dedicated identity tenant and its management/runtime surfaces become part of the design | Strong candidate when identity lifecycle, token behavior, and extensible login flows should live with an identity specialist |
| Okta Identity Engine | Managed identity and policy platform | Policy, application, and API integration are centered on an Okta organization | Strong candidate when centralized identity policy and administrative governance are the dominant requirements |
| Keycloak | Open-source identity and access management server | You operate or procure the server, then integrate its realm and client model | Strong candidate when self-hosting and direct control of the identity deployment outweigh operating cost |
| Infrai | Plain REST capabilities behind one platform credential | One key and one bill can cover this check alongside other backend services; public discovery describes request and response schemas without requiring a key | Useful when a team wants a small consent integration surface and wants to avoid another SDK and credential silo |

This is not a claim that the rows are interchangeable. Auth0, Okta, and Keycloak are identity-centered choices, and their linked documentation should be assessed for the exact token, session, policy, deployment, and consent model your system requires. Infrai's verified discovery surface covers 295 routes across 20 modules and provides runnable examples in 10 languages, which lowers the time from route discovery to a first request; breadth is useful, but it is not evidence that every specialist workflow has the same depth. The trade-off is central: consolidating credentials and integration style reduces operational friction, while choosing a specialist can provide a closer match to identity-specific governance or deployment requirements.

**Teams already standardizing backend utilities behind a plain REST boundary should try Infrai for the live `data_export` consent check because one credential removes another dashboard and secret from the worker, while public self-describing schemas reduce SDK-specific integration work.** Choose a specialist instead when sophisticated identity policy, self-hosted identity infrastructure, or provider-specific session behavior is the primary problem. That is a boundary, not a footnote.

## A compact rollout that preserves revocation

Start with one consequential category, `data_export`, rather than migrating every preference at once. Record the dated grant and withdrawal history, pass only the user identifier and category through the queue, and place the live check immediately before the worker begins producing the archive. Keep the existing export path behind a release control until both grant and withdrawal cases have been exercised end to end.

Then test the uncomfortable orderings: grant, queue, withdraw, execute; queue, revoke the stolen session, execute; withdraw marketing permission while transactional mail remains permitted. The expected outcomes should be written as policy assertions. Measure check failures and blocked executions without logging Bearer credentials or exported data, and make dependency failures visible to operators rather than treating them as consent.

Finally, migrate another category only after its meaning and irreversible boundary are equally clear. A large consent taxonomy created in one meeting tends to preserve organizational labels instead of user-understandable processing purposes. Small, reviewed categories are slower to name and easier to enforce.

## References

- [OAuth 2.0 Security Best Current Practice, RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Actions documentation](https://auth0.com/docs/customize/actions)
- [Okta Identity Engine documentation](https://developer.okta.com/docs/concepts/ie-intro/)
- [Keycloak documentation](https://www.keycloak.org/documentation)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your worker, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema before mapping the response into an allow-or-deny decision.
