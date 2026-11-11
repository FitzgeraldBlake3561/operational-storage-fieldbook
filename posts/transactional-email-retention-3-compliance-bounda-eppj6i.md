# Transactional Email Retention: 3 Compliance Boundaries for Welcome Report Service APIs

For welcome emails carrying student reports, choose a transactional email service API whose compliance evidence survives attachment expiry. Start with one generated report object and a separate send ledger. The bill to examine includes report generation, stored report copies, message submission, retained events, and the engineering time spent reconciling them; for an attachment-heavy workflow, quantify the retained bytes before arguing about email rates.

Consider an illustrative batch of 12,000 welcome reports, each exactly 1 MB, retained for 30 days. The report payload alone accounts for 360 GB-days of storage, using decimal units and excluding metadata, versions, backups, and message encoding. Keeping a second full copy doubles that payload term. These are planning assumptions, not measured traffic or a provider quote; whether storage dominates the actual bill still depends on the contract and the workload.

TL;DR: Compare transactional email services by the compliance evidence their APIs and event records let you preserve. For an edtech welcome email carrying a generated report, separate three retention boundaries: report bytes, delivery evidence, and the recipient's copy. Reducing the illustrative report retention from 30 days to 7 days changes the payload term from 360 to 84 GB-days, but only adopt that window if the evidence and recovery requirements permit it.

## What compliance evidence should a transactional email service retain for welcome emails?

The answer determines the storage design. A report object answers what content was available; a send ledger answers what the application attempted; a delivery event answers what the mail system reported. Treating those as interchangeable produces an attractive dashboard and a weak explanation when a parent disputes receipt.

An API acceptance response belongs in the ledger, but the application should not interpret it as proof that a parent received, opened, or understood the report. The official email documentation listed below distinguishes sending from subsequent delivery and bounce handling. Preserve that distinction in application states and in support language, especially when a welcome message also grants access to student information.

Keep evidence separate.

For this design, define a report version, an internal send identifier, the intended recipient reference, the selected processing location, a provider message identifier when available, and timestamps for submission and subsequent events. Store the policy version that authorized the retention deadline. These are proposed fields, not a universal compliance schema: legal and security reviewers must decide which fields establish the required evidence, which contain personal data, and how long each may remain. A recipient reference still needs access controls and its own deletion policy. A digest can help compare retained bytes with an earlier recorded value, but it does not recover a deleted attachment, demonstrate receipt, or independently establish the truth of the ledger. That last boundary matters to a storage architect: an audit claim is only as defensible as the records and controls supporting it. A long retention interval cannot repair an ambiguous event definition.

EU/US compliance therefore needs a written acceptance criterion before service selection. Ask what must be demonstrated about processing location, recipient data, event retention, access, and deletion, and obtain the relevant contractual and technical evidence. A geographic setting alone is insufficient evidence for that whole decision. This note does not assert that a particular deployment or retention window satisfies a legal obligation.

## Compare the three retention boundaries before comparing APIs

The useful comparison is a testable account of what each candidate service lets the application observe and control. Sending API convenience matters, but a convenient submission call leaves the evidence problem unresolved if support cannot connect a bounce to the student report version that triggered it.

| Boundary | Proposed retained material | Evidence to request or test | Deliberate trade-off |
| --- | --- | --- | --- |
| Generated report | One object version until its approved expiry | Version identity, access records, deletion behavior, and any separately retained copies | Shorter retention reduces stored payload but limits exact-content investigation |
| Submission and events | A minimized send ledger plus event records under a separate policy | Message correlation, event definitions, authentication, retention, and export behavior | Longer evidence retention supports reconciliation but creates another data lifecycle |
| Recipient attachment | Record that the message included the specified report version | What the sending system can actually establish about submission and delivery | Application-side expiry does not establish deletion of a recipient's copy |

Do not make the object identifier double as the send identifier. A report may need more than one authorized submission, and conflating those identities makes it difficult to explain a resend without implying that the content changed. Conversely, a newly generated report should have a distinguishable version even when its human-facing filename remains the same.

Custom-domain support belongs in this comparison as an identity requirement.

The official sending documentation covers identity verification and domain authentication; request evidence that the intended sender identity is configured, then verify the resulting message in a controlled test. Keep sender authentication evidence distinct from processing-location evidence. Neither answers the other's question.

For bounce handling, specify the decision the application needs to make. A permanent failure should enter a review or suppression path under the application's policy; an ambiguous submission result needs reconciliation before another send. Demand documented event semantics and a demonstrable correlation path. Do not infer either from an API's naming style.

This is also where team costs become visible.

Someone must own event ingestion, changes to the event schema, expired-object support requests, and evidence exports. A service comparison that omits those responsibilities has left a substantial part of the implementation unpriced, even if the message charges are perfectly accurate.

## Make the retention calculation inspectable

A retention proposal should expose its assumptions in a form a reviewer can change. The small Python example below models only the report payload for the illustrative welcome batch; it deliberately accepts a copy count because a retained duplicate has the same payload size even when its operational purpose differs.

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class ReportRetention:
    reports: int
    bytes_per_report: int
    days: int
    copies: int = 1

    def payload_gb_days(self) -> Decimal:
        values = (self.reports, self.bytes_per_report, self.days, self.copies)
        if any(value < 0 for value in values):
            raise ValueError("Retention inputs must be nonnegative")
        return (
            Decimal(self.reports)
            * Decimal(self.bytes_per_report)
            * Decimal(self.days)
            * Decimal(self.copies)
            / Decimal(1_000_000_000)
        )


illustrative_policies = {
    "student_welcome_report_30_days": ReportRetention(12_000, 1_000_000, 30),
    "student_welcome_report_7_days": ReportRetention(12_000, 1_000_000, 7),
}

for policy_name, policy in illustrative_policies.items():
    print(policy_name, policy.payload_gb_days())
```

The calculation gives 360 and 84 GB-days.

It isn't a complete invoice estimator: it omits submission charges, generation, event storage, encoding, transfer, requests, and operational effort. Its purpose is narrower and useful: expose the retained-payload term so that a reviewer can challenge the report size, the number of copies, or the proposed lifetime without accepting an unexplained total.

Before changing retention in production, compare that model with an inventory of actual report versions and separately retained copies. A deadline recorded in a database is an instruction, not evidence that every retained copy has disappeared. Define how deletion completion is observed and what happens to a support investigation when the bytes have already expired.

Test the policy in staging with synthetic student reports. Include an ordinary submission, a permanent bounce, repeated event delivery, an event that arrives after report expiry, and an interrupted submission whose final status is initially unknown. These are proposed failure tests, not claims about the frequency or behavior of any named service. The expected outcome should describe the ledger transition and evidence retained, rather than merely require a successful API response.

For deployment, assign an owner to retention changes and review the expiry policy alongside the send workflow. Observe unresolved submissions, unmatched events, deletion failures, and events arriving after payload expiry. Minimize the personal data in diagnostic logs; otherwise the troubleshooting path quietly becomes an additional report archive.

## Stop keeping report bytes when the evidence no longer needs them

The awkward failure mode is the uncertain send. If the process loses its response after submission, an immediate retry can produce a duplicate attachment, while declaring success can leave an unexplained gap. Record an explicit unknown state and reconcile against the available message evidence. Where a candidate offers deduplication or lookup controls, test their documented boundaries before depending on them; this design does not assume every API provides those controls.

A second trap appears when welcome emails carry account-recovery material as well as reports. Keep the two purposes distinct. The OWASP guidance in References calls for recovery tokens to be securely generated, single use, and time limited. A retained mail event is not a reason to retain a usable recovery secret in the ledger or in diagnostic output.

Approve the service only when the team can show the sender identity, trace a report version through submission and bounce handling, explain the relevant EU/US processing evidence, and apply the agreed retention policy.

A product ranking cannot substitute for those acceptance conditions.

For the illustrative 7-day policy, deliberately stop keeping the original report bytes after day 7, including any copies covered by that policy, while retaining only the separately approved evidence fields. The cost is concrete: a later complaint may be traceable to a submission and its events, yet the team cannot inspect or resend the exact original attachment from that archive. Regenerating from changed student data may produce different content. If exact replay is required, retain an approved immutable version for the required interval and account for its storage explicitly.

Deletion narrows recovery. That is the decision to sign off, alongside the evidence that remains.

## References

The official sending documentation provides the technical basis for distinguishing submission, identity configuration, delivery, and bounce handling. The security guidance supports the separate treatment of recovery tokens. Neither source establishes a complete EU/US legal compliance determination, and the batch sizes and retention windows above are explicitly illustrative design inputs.

## Further reading

- Amazon SES official documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
