# Testing & Validation

This document describes how the AI-Powered Freelance Opportunity Aggregator was tested and calibrated during development.

Validation covered both technical execution and decision quality.

The goal was not only to confirm that workflows completed successfully, but also to verify that filtering, semantic evaluation, classification, delivery, and deduplication behaved as intended on real opportunity data.

## Validation Strategy

Testing was performed iteratively throughout development using repeated workflow executions and real opportunity records.

The main validation areas were:

- deterministic hard-filter behavior;
- semantic AI evaluation quality;
- structured-output consistency;
- deterministic Decision Engine behavior;
- delivery prioritization;
- duplicate prevention;
- end-to-end workflow execution;
- LLM usage and operating cost.

Testing focused particularly on false negatives.

For this system, incorrectly discarding a potentially relevant opportunity was considered more costly than allowing an uncertain opportunity to continue to manual review.

> **Prefer review over premature rejection when the available evidence is ambiguous.**

---

## Hard Filter Validation

The Hard Filter was tested against real opportunity batches rather than only synthetic examples.

Its purpose was deliberately limited to clear hard failures.

During testing, several edge cases revealed rules that were initially too aggressive.

### Seniority False Positives

Early versions of the filter sometimes interpreted words associated with leadership or seniority outside their actual context.

For example, terms that contained words such as `lead` could incorrectly trigger seniority-related rejection even when they referred to activities rather than job level.

The rule was refined so that seniority blocking depended on stronger contextual evidence rather than isolated keywords.

### Compensation Negation

Compensation-related filtering also required contextual handling.

A naive keyword rule could incorrectly reject a listing containing compensation-related language even when the text explicitly negated the problematic condition.

The rule set was updated to account for negation rather than treating the presence of a keyword as sufficient evidence.

### Geographic Eligibility

Geographic restrictions remained one of the strongest deterministic blockers.

Testing verified that explicit incompatibility could produce a hard failure while missing or ambiguous geographic information was allowed to continue to semantic evaluation.

### Incomplete Listings

Listings with insufficient or incomplete information were intentionally allowed to pass unless a confirmed hard blocker was present.

This reduced the risk of losing useful opportunities simply because the original listing omitted details.

### Batch Validation

A larger Hard Filter validation run processed:

- **135 opportunities**
- **92 PASS**
- **43 FAIL**

The purpose of this run was not to maximize the number of rejected records, but to confirm that failures were supported by explicit hard-filter reasons.

---

## AI Evaluation Calibration

The semantic evaluation layer required repeated calibration because technically valid structured output can still contain poor reasoning.

Testing therefore focused on the quality of the model's interpretation rather than only schema compliance.

### Hallucinated Skill Gaps

One recurring issue was the creation of unsupported skill gaps.

The evaluator could sometimes infer requirements such as infrastructure, model training, backend engineering, or specialized machine-learning tooling even when the opportunity description did not explicitly require them.

The prompt was refined around a stricter evidence rule:

> A skill gap should only be reported when the specific opportunity provides evidence that the skill is materially required.

Generic weaknesses should not be inferred merely because a skill is absent from the candidate profile.

### Hypothetical Requirements

The evaluator was also instructed not to convert hypothetical requirements into real blockers.

For example, generic references to AI or Python should not automatically imply requirements for:

- model training;
- MLOps;
- deep-learning frameworks;
- production ML infrastructure;
- unrelated backend responsibilities.

This reduced unnecessary `REVIEW` and `SKIP` outcomes caused by imagined requirements.

### Source-Related Bias

Source metadata and source-specific skill tags were initially available to the evaluator.

Testing showed that these signals could influence semantic judgment even when the actual opportunity text did not justify the inference.

To reduce this bias, source identifiers, URLs, and platform-generated skill tags were removed from the LLM evaluation input.

The evaluator now operates primarily on normalized opportunity content and candidate-fit context.

### Uncertainty Calibration

Uncertainty was separated from confirmed skill gaps and confirmed blockers.

The evaluator was instructed to report significant uncertainty only when an unresolved fact could materially affect the decision.

A missing requirement is not automatically uncertainty.

A candidate skill gap is also not automatically uncertainty.

This distinction helps prevent the model from using `uncertainty` as a generic fallback whenever information is incomplete.

---

## Structured Output Validation

The semantic evaluator returns a predefined structured schema rather than free-form prose.

Testing verified that downstream logic receives predictable fields for:

- scope status and confidence;
- evaluation scores;
- semantic blockers;
- uncertainty;
- strengths;
- skill gaps;
- summary.

The schema allows the Decision Engine to operate without parsing or interpreting natural-language reasoning.

This creates a clear boundary:

```text
LLM
    ↓
Structured semantic assessment
    ↓
Deterministic program logic
```
The LLM interprets the opportunity.
The application decides what to do with that interpretation.

## AI Evaluation Provenance Validation

Evaluation persistence was extended to retain the provenance of each semantic assessment.

Stored metadata now includes:

- evaluation version;
- model name;
- prompt version;
- raw structured model response;
- evaluation timestamp.

Testing verified that the raw structured response is persisted as JSON rather than as an encoded text string and remains separate from the deterministic `total_score` and final `decision`.

This allows the original model interpretation to be inspected independently from the Decision Engine result and makes later prompt or model comparisons auditable.

## AI Failure Fallback Testing

The AI Evaluation workflow was explicitly tested against a forced evaluation failure.

A controlled invalid prompt expression was introduced to cause the semantic evaluation node to fail before producing a valid structured result.

The expected error path was:

```text
AI Evaluation
    ↓ error
Fallback Transformation
    ↓
REVIEW
    ↓
Evaluation Persistence
```

The fallback produced a valid evaluation object with:

- `scope_status = uncertain`;
- low scope confidence;
- significant uncertainty;
- zero placeholder scores;
- the original evaluation error stored in `raw_response`;
- `decision = REVIEW`.

This verified that malformed or failed AI output does not silently disappear and cannot automatically become a `SKIP`.

After the test, the production prompt was restored and the fallback path remained connected as the permanent failure-handling behavior.

## Decision Engine Verification

The Decision Engine was tested independently from the LLM.

This was important because the final `APPLY`, `REVIEW`, or `SKIP` classification is produced by deterministic JavaScript rules rather than by the model.

Validation covered combinations of:
- scope status;
- scope confidence;
- high-confidence semantic blockers;
- significant uncertainty;
- total evaluation score;
- skill-match score;
- gap-manageability score.

The tested behavior includes:
```
High-confidence out-of-scope
        → SKIP

High-confidence semantic blocker
        → SKIP

Uncertain scope
        → REVIEW

Significant uncertainty
        → REVIEW

Strong score
+ sufficient skill match
+ manageable gaps
        → APPLY

Moderate score without a hard blocker
        → REVIEW
```
The important property is determinism:

**The same structured evaluation input should always produce the same final classification.**

This makes final decisions easier to audit and prevents response variability in the LLM from directly controlling downstream actions.

## Batch Processing Validation
AI Evaluation is processed in bounded batches.

The production evaluation query is limited to:
```
20 opportunities per run
```
Testing confirmed that this limit provides a predictable upper bound for:
- LLM calls;
- workflow execution time;
- backlog processing;
- operating cost.

The batch query also excludes opportunities that already have an evaluation for the current evaluation version.

This prevents unnecessary repeated evaluation while allowing older records to be reprocessed when a new evaluation version is introduced.

## Delivery Prioritization Testing
The Telegram delivery workflow was tested separately from evaluation.

The delivery query considers only:
```
APPLY
REVIEW
```
`APPLY` opportunities are ordered before `REVIEW`.

There is intentionally no fixed quota reserved for `REVIEW`.

If at least 10 undelivered `APPLY` opportunities are available, the digest may contain 10 `APPLY` opportunities.

The digest is limited to:
```
10 opportunities per delivery
```
This limit is intended to control human information load rather than model usage.
## Delivery Deduplication Testing
Delivery deduplication was validated through repeated workflow executions.

After a digest is successfully sent, each delivered evaluation is written to:
```
public.opportunity_delivery_log
```
Subsequent selection queries join against the delivery log and exclude evaluations already delivered through the Telegram channel.

Testing confirmed that:
1. selected evaluations were delivered;
2. successful deliveries were recorded;
3. rerunning the selection query excluded those evaluations;
4. previously delivered items remained preserved in delivery history.

This means delivery deduplication is based on persisted state rather than temporary workflow memory.

## End-to-End Validation
The complete Daily Orchestrator was tested as one production flow.

The test executed the full sequence:
```
8 Discovery Workflows
        ↓
Merge / Synchronization
        ↓
Hard Filter
        ↓
AI Evaluation
        ↓
Decision Engine
        ↓
Telegram Daily Digest
        ↓
Delivery Log
```

The end-to-end run successfully:
- completed all discovery branches;
- synchronized downstream processing;
- filtered newly discovered opportunities;
- evaluated the eligible batch;
- generated deterministic decisions;
- selected the highest-priority undelivered results;
- delivered 10 opportunities through Telegram;
- recorded the delivery state.

The delivered batch contained 10 `APPLY` opportunities because enough high-priority items were available to fill the digest.

This also confirmed that delivery prioritization behaved as designed rather than enforcing an artificial `REVIEW` quota.

## Discovery Fault-Isolation Testing

The Daily Orchestrator was tested with a deliberately invalid discovery workflow reference.

One discovery call was temporarily configured to target a nonexistent workflow while the remaining source-specific discovery calls retained their normal configuration.

The failing branch returned an explicit workflow error, but the parent Daily Orchestrator continued processing instead of terminating.

This test verified that:

1. one source-specific discovery failure does not stop the remaining discovery workflows;
2. the failed branch still produces a completion result for synchronization;
3. downstream processing can continue against data persisted by the successful sources;
4. the source failure remains visible for troubleshooting.

The original discovery workflow reference was restored after the test.

This behavior was also validated against real transient source failures during scheduled operation, where source-specific errors could be corrected without redesigning the downstream pipeline.

## Operational Workflow Validation

Two independent support workflows were added and tested separately from the daily production pipeline.

### DB Housekeeping

The database housekeeping workflow runs on a weekly schedule.

Before deletion, it performs a count query for opportunities older than the configured retention window of 180 days.

An IF condition allows deletion to run only when the count is greater than zero.

During validation, the count returned zero records eligible for deletion, so the workflow correctly followed the non-destructive branch and no data was removed.

The deletion query itself is designed to return the number of deleted rows when eligible records exist.

### Weekly SKIP Audit

A separate weekly quality-control workflow samples opportunities classified as `SKIP`.

The audit first searches for potentially suspicious false negatives:

- opportunities with `in_scope` status or relatively high scores;
- without any confirmed high-confidence semantic blocker.

Up to three such borderline records are selected.

The remaining audit capacity is filled with random `SKIP` records until the sample contains up to five opportunities.

The resulting sample is formatted into a Telegram digest showing:

- opportunity title;
- total score;
- scope classification;
- actual reason for `SKIP`;
- source URL.

When a `SKIP` is caused by a high-confidence semantic blocker, the digest reports the blocker reason rather than only the general scope explanation.

End-to-end testing confirmed successful SQL selection, formatting, and Telegram delivery.

In the tested production sample, no true borderline false negatives were detected, and the remaining random `SKIP` decisions were consistent with the intended candidate profile.

## Cost Validation
Cost efficiency was treated as an architectural requirement.

A complete end-to-end production run cost approximately:
**$0.06**

The system avoids sending every discovered opportunity directly to the LLM.

Instead:
```
Discovery
    ↓
Normalization
    ↓
Deduplication
    ↓
Hard Filter
    ↓
Bounded AI Evaluation
```
This keeps semantic processing focused on opportunities that have already passed inexpensive deterministic checks.

## What Was Validated
By the end of the production testing cycle, the following behaviors had been verified:
- heterogeneous discovery workflows can operate under one shared data contract;
- repeated discovery runs do not create unnecessary duplicates;
- deterministic filtering avoids rejecting ambiguous opportunities without clear evidence;
- the LLM produces structured semantic assessments rather than final actions;
- unsupported skill gaps and hypothetical requirements are reduced through prompt calibration;
- the Decision Engine produces deterministic classifications;
- evaluation versioning supports controlled reprocessing;
- Telegram delivery prioritizes higher-value opportunities;
- successfully delivered evaluations are not repeatedly shown;
- the full pipeline can execute end-to-end on a scheduled production workflow;
- source-specific discovery failures are isolated and do not terminate the full daily pipeline;
- AI evaluation provenance is persisted for later debugging and comparison;
- malformed or failed AI evaluations fall back to REVIEW rather than silent rejection;
- weekly database maintenance and SKIP quality auditing operate independently from the daily production workflow;
- bounded LLM usage keeps operating cost low.

## Validation Philosophy
The project follows a simple testing principle:

**A successful workflow execution is not enough. The resulting decisions must also behave correctly.**

This distinction was particularly important for the AI evaluation layer.

A green execution state confirms technical completion.

Validation of reasoning quality, uncertainty handling, filtering behavior, and final classification confirms whether the system is actually useful.
