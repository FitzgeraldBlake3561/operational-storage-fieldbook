# Python Moderation: Reduce LLM Cost as Small Models Classify Reports into JSON

To reduce LLM cost when small models summarize developer-tool reports, classify them, and extract JSON, start with the bill's components: input tokens, generated tokens, retries, retained evidence, and human review. A useful first calculation is an accounting identity: if a workflow sends the same report through three stages that each read the full report, its report-body input is three times the body input of one stage. This is arithmetic for that workflow, not a measured production result; different instructions, output lengths, and retry rates can change which term dominates the actual bill.

Short answer: reduce LLM cost by measuring repeated input first, then test a single structured classification against the existing summarize, classify, and extract JSON chain. Admit a smaller model only when its results satisfy both the output contract and the moderation policy. Batch processing is a scheduling choice. Retention is a separate decision about how much evidence remains available when a reviewer challenges the classification.

## How can small models reduce LLM cost when they classify reports?

For this design, a moderation report concerns content in a developer collaboration tool: a reported comment, its immediate context, and a reporter-supplied reason. The model proposes a category before human review; it does not delete content or sanction an account. That boundary matters because a valid category can still be wrong, and a polished summary can quietly discard the evidence that a reviewer needs.

Start with a ledger at the report level. Attribute input tokens, output tokens, attempts, and final disposition to each stage, using the serving system's usage accounting and a tokenizer appropriate to the selected model for preflight estimates. Record estimated and billed usage separately so that an estimate never silently becomes an invoice claim.

Consider a deliberately simplified workload assumption: 1,000 reports, each containing 800 report-body tokens, pass through three stages. That means 2,400,000 report-body input tokens before instructions or retries; passing the body once would contribute 800,000 instead. These figures describe only repeated body input. They do not establish a monetary saving, and they do not include the extra output that a combined operation might require. The point of the example is to identify a term worth measuring, then check whether it really dominates the observed workload.

Measure before merging.

A combined request can propose the category and extract the evidence reference in one response, while a human-facing summary can be generated only if the review interface actually needs it. This removes one reason to repeat the full context. The trade-off is coupling: if a combined response fails validation, retrying it may repeat work that separate stages could have preserved. Keep the original design as the comparison until the ledger shows fewer total tokens per accepted, correctly classified report.

## Treat valid JSON as the first boundary

JSON syntax, an application schema, and moderation correctness are three different checks. RFC 8259 defines the interchange format; JSON Schema provides vocabulary for structural constraints; the moderation policy determines whether the selected category is justified. Passing the first two does not establish the third.

For a concrete application contract, let a proposal contain exactly four fields: a report identifier, one category from a versioned taxonomy, an evidence identifier, and a human-review flag that must be true. The categories below are example application choices, not a universal moderation standard. An evidence identifier must resolve to a retained excerpt for that same report; accepting an arbitrary string would leave the most important integrity check unfinished.

The Python example validates an already decoded proposal against trusted request context. The retained-evidence mapping belongs to the application and must be authorized for the current report before it reaches this function.

```python
CATEGORIES = frozenset({
    "harassment", "spam", "other", "needs_context"
})
FIELDS = frozenset({
    "report_id", "category", "evidence_id", "human_review"
})


def validate_proposal(proposal, expected_report_id, retained_evidence):
    if type(proposal) is not dict or set(proposal) != FIELDS:
        raise ValueError("Unexpected proposal fields")
    if type(proposal["report_id"]) is not str:
        raise ValueError("Report ID must be a string")
    if proposal["report_id"] != expected_report_id:
        raise ValueError("Proposal belongs to another report")
    category = proposal["category"]
    if type(category) is not str or category not in CATEGORIES:
        raise ValueError("Unknown moderation category")
    evidence_id = proposal["evidence_id"]
    if type(evidence_id) is not str or evidence_id not in retained_evidence:
        raise ValueError("Evidence is unavailable for this report")
    if proposal["human_review"] is not True:
        raise ValueError("Human review is required")
    return dict(proposal)


proposal = {
    "report_id": "report-1042",
    "category": "needs_context",
    "evidence_id": "excerpt-7",
    "human_review": True,
}
validated = validate_proposal(
    proposal,
    expected_report_id="report-1042",
    retained_evidence={"excerpt-7": "Reported comment and adjacent reply"},
)
```

This catches missing fields, unexpected fields, an invented category, and a reference to evidence outside the supplied set. It cannot tell whether an excerpt proves harassment. It also starts after decoding: the ingestion boundary still needs a body-size limit, a parser policy for duplicate object names, and rejection of malformed responses. RFC 8259 says object names should be unique and describes interoperability problems when they are not; silently accepting whichever duplicate a decoder preserves makes a poor audit contract.

There is a quieter failure mode here. A model can cite an allowed excerpt that does not support its category, so the evaluation set must test evidence relevance alongside category correctness. Report text is also untrusted input: instructions embedded in a reported comment must not become permission to change the output contract. Application validation remains necessary even when the generation interface offers structural constraints.

## Compare models and batching against the review contract

The useful comparison is between candidate configurations on the same held-out reports, taxonomy, and acceptance rules. A smaller model, a larger model, and a routed combination are hypotheses to test; parameter count alone cannot establish which one handles ambiguous developer conversations correctly. Keep difficult cases visible, including quoted abuse, missing thread context, conflicting reporter descriptions, and attempts to insert instructions into the content being classified.

| Decision | What to measure | Trade-off to retain in the decision |
| --- | --- | --- |
| One combined classification request | Total tokens per correctly accepted report, including retries | Less repeated context can mean more work repeated after a failed combined response |
| Smaller candidate model | Contract acceptance, category errors, and evidence relevance | A lower request bill is insufficient if review effort rises |
| Escalation after abstention or validation failure | Escalation frequency and total attempts | A second attempt can erase the benefit of the first |
| Asynchronous batches | Completion delay, duplicates, and missing results | Scheduling flexibility requires reconciliation before review |
| Shorter evidence retention | Retained volume and unresolved appeals | Less stored content leaves fewer facts available for later investigation |

Do not collapse these measures into a single JSON success percentage. Count malformed responses, structurally valid but unsupported classifications, and explicit abstentions separately; otherwise a model that emits tidy objects can appear successful while sending misleading labels to reviewers. Evaluate the whole routing path as well as individual models, because fallback requests consume tokens too.

Batching should follow the review deadline. If a report must reach a reviewer immediately, deferred completion may be the wrong contract regardless of request economics; if a queue can tolerate delay, asynchronous submission can be tested without changing the category schema. Measure the terms and limits of the actual execution system before assigning a monetary benefit to batching.

Reconcile by report identifier, never by array position. A missing result must remain a pending or failed job rather than inherit the next report's classification. Keep a bounded retry policy and send exhausted or undecodable jobs to human review with an explicit unavailable-proposal state. Those are proposed application behaviors, and their operational value depends on implementing and testing them.

## Retention determines what an audit can prove

The data layer should distinguish the original report, the retained evidence excerpt, and the accepted proposal. A proposed record includes their identifiers plus the taxonomy, prompt, validator, and model configuration versions, so an investigator can determine which rules produced a decision. Persisting a response without its rule versions preserves bytes while losing much of their meaning.

A content digest can help check whether an available object matches a recorded digest. It cannot reconstruct deleted content.

Define the consistency boundary before adding cleanup jobs: either an accepted proposal and its evidence reference become visible together, or a reconciliation mechanism detects and repairs the gap. Otherwise the system can present a classification whose cited evidence was never durably retained. Similarly, a retention worker must consider active review and appeal state before deleting an excerpt; a timestamp alone does not express those dependencies. These are design requirements, not claims about the guarantees of an unspecified storage service.

Consider a hypothetical appeal against report-1042. The decision record names excerpt-7, but a cleanup worker has removed that excerpt while retaining the category and its digest. Replaying the validator now rejects the reference, which is useful detection, yet it cannot restore the missing words or establish whether the original classification was justified. Keeping another generated summary would not resolve that evidentiary gap either: the summary is another interpretation, and the disputed omission might be precisely what it left out. The trade-off is therefore concrete: retain the authorized excerpt for the active appeal lifecycle, accepting its storage and access-control burden, or accept that later review can establish the recorded decision but cannot reexamine its evidence. A deployment test should exercise that transition with synthetic records: create a pending appeal, attempt expiry, and verify that the retention rule preserves the referenced evidence until the appeal reaches its terminal state. This is a proposed failure test, not an incident report or an assertion that an existing service already implements the rule.

Keep operational telemetry smaller than the evidence store. Aggregate token usage, validation failure classes, retry counts, completion lag, and reviewer overrides without copying report bodies into every log event. This reduces the number of places that need a deletion policy, although it also means an operations log alone will not explain the wording that caused a disputed classification. Test deployment changes against a fixed evaluation set, version the schema with the validator, and check that old pending jobs can still be interpreted after a rollout.

Choose a retention period with the owner of the review and appeal requirements. Storage cost belongs in the ledger, but it should not become permission to remove the only evidence supporting a moderation decision.

## Stop keeping intermediates deliberately

The decision rule is to adopt a changed pipeline only when it preserves the agreed category and evidence acceptance criteria while reducing measured total work per accepted report. A cheaper attempt that creates more retries, unsupported labels, or manual reconstruction has not demonstrated that result.

For this proposed design, stop retaining redundant intermediate summaries once the accepted proposal and required evidence are durable, and expire raw generation responses under a separately defined debugging policy. Keep the compact decision record and the evidence required for the active review lifecycle. This makes the retention boundary explicit instead of letting accidental log copies decide it.

The loss is real.

After an intermediate response expires, an investigator cannot recover its exact wording from the final category or a digest; after the underlying evidence expires, even a perfectly valid JSON record cannot establish what the original report said. The storage boundary therefore determines which future questions the system deliberately gives up answering.

## References

The format and duplicate-name discussion uses RFC 8259. Structural validation terminology follows JSON Schema; the Python reference supports the distinction between decoding JSON and applying an application contract. Workload numbers above are explicitly stated arithmetic assumptions, and the pipeline, taxonomy, and retention rules are proposed design choices.

## Further reading

- RFC 8259, The JavaScript Object Notation Data Interchange Format: https://www.rfc-editor.org/rfc/rfc8259
- JSON Schema validation specification: https://json-schema.org/draft/2020-12/json-schema-validation
- Python JSON encoding and decoding reference: https://docs.python.org/3/library/json.html
