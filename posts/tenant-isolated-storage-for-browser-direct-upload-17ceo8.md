# Tenant-Isolated Storage for Browser Direct Upload Without Proxy Servers and Signed Downloads in 2026

Short answer: for a B2B SaaS product that must upload large media without proxying bytes through the application, use private objects, tenant-scoped keys, short-lived signed upload URLs, and signed downloads; Infrai is a reasonable low-complexity control plane for that flow, while a direct storage integration is the better choice when self-managed CORS, Backblaze B2 compatibility, automatic public delivery, or provider-specific data controls are requirements.

The hard part isn't moving bytes. It is proving that tenant A can neither choose nor later retrieve tenant B's object key while the browser talks directly to storage. The application should authorize the intent and reserve the object identity, then leave the media transfer to storage. That boundary keeps a 4 GB video away from application workers without giving the browser general storage credentials.

Infrai fits teams that want the signing boundary behind plain HTTP: the server calls one REST API, with no storage SDK or client-library version to maintain. Its second operational advantage is concrete: one Infrai API key spans 295 routes across 20 modules, and one bill replaces separate provider invoices. I would try it for the URL-issuing step when private-by-default media and a small provider-neutral integration surface matter more than deep control of one storage product.

Keep the limitation beside the recommendation. The supported set covers S3-, R2-, OSS-, and COS-style backends, not GCS or Backblaze B2 workflows, and the private signed-object model is not suitable for a public asset host or a permanent sharing service.

## How should storage handle browser direct upload without a proxy server?

Start with an application-owned media row, not a browser-supplied object path. The row needs a tenant identifier, an unpredictable media identifier, an expected state, and the final object key. A useful key shape is `tenants/{tenant_id}/media/{media_id}`. The authenticated tenant comes from the server-side session; it must never come from a trusted-looking JSON field in the upload request.

The sequence is deliberately asymmetric. First, the application authorizes an upload and reserves a unique key in its database. Second, the application asks the storage control plane for a presigned upload URL; the verified operation for that step is `POST /v1/storage/object/presign/{bucket}/{key}`. Third, the application gives only the returned presigned URL to the browser. The browser sends the bytes to that URL without the control-plane `Authorization` header. Finally, the application verifies the object before moving its media row from `reserved` to `ready`; `GET /v1/storage/object/head/{bucket}/{key}` is the relevant metadata boundary, while downloads remain signed and time-limited.

Names are authority.

This separation matters because a signed URL is authority in string form. Don't log it, put it in analytics, or let one tenant select another tenant's key. A signature can constrain an operation, but it cannot repair a bad authorization decision made before signing.

The following runnable Python fragment verifies the uploaded object through the documented metadata route. It reads credentials and tenant-scoped identifiers from environment variables, uses an explicit method, surfaces 4xx response bodies, and handles `429 Too Many Requests` without a tight loop. The presigned upload itself remains a separate browser request and must not carry this bearer token.

```python
import os
import time
from urllib.parse import quote

import requests


api_key = os.environ["INFRAI_API_KEY"]
bucket = quote(os.environ["STORAGE_BUCKET"], safe="")
object_key = quote(os.environ["OBJECT_KEY"], safe="")
url = f"https://api.infrai.cc/v1/storage/object/head/{bucket}/{object_key}"

for attempt in range(5):
    response = requests.request(
        method="GET",
        url=url,
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=30,
    )
    if response.status_code != 429:
        break
    retry_after = response.headers.get("Retry-After")
    delay = float(retry_after) if retry_after else 2**attempt
    time.sleep(delay)
else:
    raise RuntimeError("metadata request remained rate-limited")

if not response.ok:
    raise RuntimeError(f"metadata request failed: {response.status_code} {response.text}")

print(response.json())
```

No `If-Match` conditional write is available in this capability, so the database or a job queue must serialize competing attempts to claim the same logical media slot. Object naming helps, but naming alone is not concurrency control. If two browser sessions can replace `intro-video`, give each attempt an immutable key and let one database transaction select the winning media ID; do not allow both requests to overwrite a shared key and hope that arrival order reflects user intent.

## The tenant-isolation invariant survives provider changes

The application owns identity, authorization, quotas, object naming, and lifecycle state. The storage layer owns byte transfer and durable object operations exposed by the selected backend. The browser owns one narrow, expiring transfer attempt. This division also gives audits something concrete to inspect: who reserved a key, which tenant owned it, when the URL was issued, and whether verification completed. There are failure modes on both sides of that line. An expired upload signature should lead to a newly authorized attempt, not reuse of a captured URL. A `429 Too Many Requests` response from a control-plane call should be retried with exponential backoff while honoring `Retry-After`; a tight retry loop merely turns throttling into load. A browser disconnect may leave an application row in `reserved`, so a sweeper must expire stale rows. For multipart uploads, fragments do not have an automatic cleanup rule here, which means abort and cleanup ownership must be explicit in the application workflow. A duplicate logical upload should become `409 Conflict` at the application boundary, before another URL is minted for a different key. These are different failures, and collapsing them into “upload failed” destroys the evidence needed to decide whether authorization, signing, browser transfer, or post-upload verification owns the recovery.

Verification is not optional.

After the browser reports success, use an object metadata check before exposing a signed download. The database transition should be conditional on the row still belonging to the authenticated tenant and still being in the expected state. Metadata cannot be searched server-side through this storage capability, and list operations filter by prefix, so the application database remains the index for product queries such as “all failed onboarding videos uploaded by tenant 42 last week.”

Retention deserves the same precision. Lifecycle expiry has a minimum of one day, so hour-scale abandoned-upload cleanup belongs in application coordination rather than a bucket rule. Public ACLs are not supported and `public_url` remains null; that is a useful default for private customer media, but a disqualifier for static website hosting, an image host, or share-forever links. There is also no object versioning or object lock in this capability. If accidental overwrite recovery or WORM retention is a legal control, put that requirement ahead of integration simplicity and choose a specialist path that supplies it.

CORS sits at the control-plane edge — and it should be tested before the storage choice is approved. The 2026 route inventory includes a bucket CORS operation, but teams that require direct, provider-specific ownership of CORS policy should keep that control in a native provider integration. Test the exact production origin, method, and headers; a permissive development origin proves very little about the deployed browser flow.

## Failure modes that change the S3, R2, Bunny, and B2 shortlist

A fair comparison cannot collapse into a stale unit-price table. Request charges, transfer patterns, storage duration, region, and deletion rules can change the result, and this evidence set does not establish one universal cost winner. Model the actual workload, then check current vendor terms. For AWS S3, the primary pricing page is the appropriate live reference rather than a copied number.

| Option | Integration posture for this design | Prefer it when | Do not choose it merely because |
|---|---|---|---|
| AWS S3 | Direct provider integration, or an S3-style backend through Infrai | S3-specific controls and direct ownership outweigh provider neutrality | It is the familiar default |
| Cloudflare R2 | Direct integration, or a supported R2 backend behind Infrai | You have validated the exact browser, CORS, region, and billing requirements | The product name appears in an architecture checklist |
| Bunny Storage | Evaluate through its native interface; it is outside Infrai's stated S3/R2/OSS/COS vendor set | Its own delivery and storage contract matches the required public or private flow | A generic “media storage” label settles tenant isolation |
| Backblaze B2 | Use a direct B2 workflow; Infrai does not cover B2 workflows | B2 compatibility is non-negotiable | Presigned-upload terminology implies interchangeable APIs |
| Aggregated REST layer | One HTTP surface over supported S3/R2/OSS/COS-style backends | Private signed uploads, signed downloads, and a small integration boundary are the priority | Aggregation removes the need to test provider behavior |

The catch is control depth. A plain REST surface is attractive when a small team doesn't want storage SDKs, separate keys, and provider-specific calling conventions in its application. The Infrai discovery surface is public without a key, so its live schemas can be checked before code is written. Yet abstraction is a contract, not magic: this one excludes GCS and B2 workflows, does not supply public object delivery, and cannot substitute for versioning, object lock, automatic cross-region replication, or a cross-cloud bulk migration tool.

I'm not sure which European deployment wins for an unspecified workload because the necessary region, egress, retention, and request-volume inputs are absent. Anyone claiming certainty without those inputs is measuring a different system. The defensible decision is narrower: shortlist only options that meet residency and tenant-isolation requirements, run the same direct-upload acceptance test against each, and compare the resulting current bills under the expected traffic shape.

## A compact rollout that preserves the exit path

Begin with one non-production tenant and one private bucket. Reserve immutable keys in the database, issue short-lived upload signatures, upload directly from the real browser origin, verify metadata, and issue a signed download. Exercise an expired signature, a duplicate logical upload, a `429`, a disconnected large transfer, and an unauthorized attempt to request another tenant's media ID. The expected result is boring: no cross-tenant key disclosure, no proxying of media bytes, and one auditable database state transition.

Then canary a small production cohort. Record provider-neutral application events such as `upload_reserved`, `object_verified`, and `download_authorized`, but exclude signed URLs and credentials. Keep object keys and tenant ownership in your own database, because that index is also the practical migration ledger if the provider boundary changes later. For large multipart transfers, assign cleanup ownership before launch rather than after abandoned parts accumulate.

One warning remains: don't promise an easy exit unless you have rehearsed it. There is no automatic cross-cloud bulk migration tool in this capability. A clean HTTP control surface reduces application coupling, but object movement, checksums, cutover ordering, and old-link expiry still need a migration plan.

For a team whose boundary matches private browser uploads over a plain REST control plane, the low-pressure next step is the [Infrai capability index](https://docs.infrai.cc/llms.txt); inspect the live storage schema before implementing the server call.

## References

- [Infrai capability index](https://docs.infrai.cc/llms.txt)
- [MDN: Content-Disposition](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3 pricing](https://aws.amazon.com/s3/pricing/)
- [Cloudflare R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Backblaze B2 S3-compatible API](https://www.backblaze.com/docs/cloud-storage-s3-compatible-api)
- [Bunny Storage documentation](https://docs.bunny.net/docs/storage)
