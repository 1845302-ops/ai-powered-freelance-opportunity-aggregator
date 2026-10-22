# Workflow Breakdown

This document describes the responsibilities and interaction boundaries of the 14 n8n workflows that make up the AI-Powered Freelance Opportunity Aggregator.

The system is divided into:

- 8 source-specific Discovery workflows;
- 1 Daily Orchestrator;
- 1 Hard Filter workflow;
- 1 AI Evaluation / Semantic Filter workflow;
- 1 Telegram Daily Digest workflow;
- 1 DB Housekeeping workflow;
- 1 SKIP Audit workflow.

Each workflow is intentionally responsible for a limited part of the system rather than combining discovery, filtering, evaluation, and delivery into one monolithic automation.

## Workflow Inventory

| Workflow group | Count | Primary responsibility |
|---|---:|---|
| Discovery workflows | 8 | Collect, normalize, and persist opportunities |
| Daily Orchestrator | 1 | Coordinate scheduled execution |
| Hard Filter | 1 | Apply deterministic exclusion rules |
| AI Evaluation / Semantic Filter | 1 | Perform structured semantic evaluation and deterministic classification |
| Telegram Daily Digest | 1 | Select, deliver, and log prioritized opportunities |
| DB Housekeeping | 1 | Perform scheduled retention checks and controlled cleanup |
| SKIP Audit | 1 | Sample rejected opportunities for manual false-negative review |

---

## Discovery Workflows

The eight Discovery workflows implement source-specific collection logic while exposing the same internal contract to the rest of the system.

Individual sources may use different ingestion mechanisms, including:

- structured APIs;
- feeds;
- structured web data;
- email-based notifications.

Despite these differences, every Discovery workflow is responsible for producing the same normalized opportunity representation before persistence.

A typical Discovery workflow performs the following stages:

```text
Source Input
    ↓
Extract Relevant Records
    ↓
Normalize Fields
    ↓
Generate Deduplication Identity
    ↓
Upsert into PostgreSQL
```

Discovery workflows may also include source-specific resilience controls such as bounded retries, batching, dynamic resource addressing, or tolerant error handling where the external source is prone to transient failures.

These controls remain local to the source-specific workflow so that integration-specific recovery behavior does not leak into the shared filtering or evaluation layers.

### Discovery Responsibilities

Each Discovery workflow is responsible for:
- retrieving available opportunity data;
- extracting relevant fields from the source representation;
- mapping source-specific values into the normalized opportunity schema;
- handling missing or inconsistent fields safely;
- generating or preserving a stable deduplication identity;
- writing normalized records into `public.opportunities`;
- avoiding unnecessary duplicate creation during repeated discovery runs.

The Discovery layer does not perform semantic candidate-fit evaluation.
Its responsibility ends once a usable normalized opportunity has been persisted.
This keeps source integration logic independent from the filtering and evaluation layers.

## Daily Orchestrator

The Daily Orchestrator is the production entry point for scheduled execution.

It coordinates the eight Discovery workflows in parallel and then triggers the shared processing workflows in sequence.
```
Schedule Trigger
      ↓
8 Discovery Workflows
      ↓
Merge / Synchronization
      ↓
Hard Filter
      ↓
AI Evaluation
      ↓
Telegram Daily Digest
```

The Discovery branches converge through a Merge node configured for append behavior.

Because each Discovery workflow already persists its results to PostgreSQL, the Merge node is primarily used to ensure that all discovery branches have completed before downstream processing begins.

The Orchestrator therefore manages execution order rather than acting as the primary transport layer for opportunity data.
Core sub-workflows are configured to wait for completion before the next processing stage begins.

Each discovery call is also configured as an independent failure boundary.

If a source-specific discovery workflow fails, the parent Orchestrator continues execution instead of terminating the full daily run. The error remains available for inspection, while successful discovery branches can complete and persist their data normally.

This allows downstream processing to continue against the successfully persisted opportunity state even during a partial source outage.

## Hard Filter Workflow

The Hard Filter performs deterministic screening before semantic AI evaluation.
Its purpose is not to determine whether an opportunity is an ideal match.
Instead, it removes only opportunities that contain sufficiently clear hard-fail conditions.

Examples include:
- explicit geographic incompatibility;
- clearly incompatible work format;
- closed or unavailable opportunities;
- unacceptable compensation conditions;
- explicit requirements that create a confirmed hard mismatch.

Ambiguous or incomplete opportunities are intentionally allowed to continue.

The filter follows the principle:
**Prefer additional review over prematurely discarding a potentially relevant opportunity.**

### Processing Flow
```
Select Unprocessed Opportunities
        ↓
Apply Deterministic Rules
        ↓
Assign PASS / FAIL
        ↓
Store Filter Status and Reasons
```
Filter results are written back to the opportunity record together with the filter version and processing timestamp.
Versioning allows updated filter logic to distinguish records processed by older rule sets.

## AI Evaluation / Semantic Filter Workflow

The AI Evaluation workflow processes opportunities that passed the current Hard Filter version and have not yet received an evaluation for the current AI evaluation version.

The workflow is intentionally separated into semantic interpretation and deterministic classification.
```
Select Evaluation Batch
        ↓
Structured LLM Evaluation
      ↙           ↘
 Success          Error
   ↓                ↓
Structured       Fallback
Output Parser      REVIEW
   ↓                ↓
Decision Engine     │
      ↘            ↙
       Persist Evaluation
```

### Evaluation Batch Selection

Processing is performed in bounded batches rather than evaluating every eligible opportunity in one execution.

This helps control:
- LLM usage;
- execution time;
- unexpected backlog size;
- daily operating cost.

### Structured Semantic Evaluation

The LLM evaluates the opportunity against a defined candidate profile.

The model returns structured fields describing areas such as:
- task relevance;
- skill match;
- gap manageability;
- role fit;
- delivery fit;
- growth value;
- semantic blockers;
- uncertainty;
- strengths;
- relevant skill gaps.

The LLM does not directly return the final operational decision.

### Structured Output Parser

A structured-output schema constrains the model response into a predictable data format.
This prevents downstream workflow logic from depending on free-form natural-language responses.

### Decision Engine

The parsed evaluation is passed to a deterministic JavaScript decision layer.

The Decision Engine considers:
- scope status and confidence;
- high-confidence blockers;
- significant uncertainty;
- total evaluation score;
- minimum skill-match requirements;
- minimum gap-manageability requirements.

It produces one of three decision-support states:
```
APPLY
REVIEW
SKIP
```
Because this stage is deterministic, identical structured evaluation input produces the same final classification.

### Persistence

The completed evaluation is stored separately from the original opportunity.
Evaluation records include their evaluation version, making it possible to test or deploy newer evaluation logic without overwriting previous results.

Evaluation persistence includes provenance metadata in addition to the semantic fields and deterministic decision.

Stored metadata includes:

- evaluation version;
- model name;
- prompt version;
- raw structured model response;
- evaluation timestamp.

The raw structured response is captured before deterministic classification, allowing the model interpretation and the Decision Engine result to be audited independently.

If the LLM or structured-output parsing fails, the workflow routes the error through a fallback transformation that creates a valid `REVIEW` evaluation with explicit uncertainty metadata rather than dropping or rejecting the opportunity.

## Telegram Daily Digest Workflow

The Telegram Daily Digest workflow is responsible for selecting and delivering the highest-priority evaluated opportunities that have not already been delivered.

Only `APPLY` and `REVIEW` opportunities are eligible for the digest.

`APPLY` opportunities receive priority over `REVIEW`.

The digest is capped to avoid information overload.
### Delivery Flow
```
Select Undelivered Candidates
        ↓
Prioritize APPLY / REVIEW
        ↓
Build Telegram Message
        ↓
Send Digest
        ↓
Prepare Delivery Records
        ↓
Persist Delivery Log
```
Each digest entry may include:
- opportunity title;
- decision state;
- evaluation score;
- compensation information when available;
- work format or location;
- concise evaluation summary;
- opportunity link.

### Delivery Deduplication

Successful deliveries are stored in `public.opportunity_delivery_log`

Future selection queries exclude evaluations already delivered through the same channel.

This prevents repeated recommendations without deleting historical delivery information.

## DB Housekeeping Workflow

The DB Housekeeping workflow runs independently from the daily production pipeline.

Its purpose is to enforce a controlled retention policy for stale opportunity data.

The workflow performs:

```text
Weekly Schedule
      ↓
Count Opportunities Older Than Retention Window
      ↓
IF rows_to_delete > 0
      ↓
Delete Eligible Opportunities
```

The production retention window is currently 180 days.

A pre-deletion count prevents unnecessary destructive execution when no records are eligible for cleanup.

Keeping housekeeping independent from the Daily Orchestrator ensures that maintenance failures cannot interrupt daily discovery, filtering, evaluation, or delivery.

## SKIP Audit Workflow

The SKIP Audit workflow provides a lightweight quality-control loop for deterministic and semantic rejection behavior.

It runs independently on a weekly schedule.

The audit prioritizes potentially suspicious `SKIP` decisions that:

- are classified as `in_scope` or have relatively high scores;
- do not contain a confirmed high-confidence semantic blocker.

Up to three such borderline records are selected.

The remaining audit capacity is filled with random `SKIP` records until the sample contains up to five opportunities.

The selected records are formatted into a Telegram digest showing the opportunity title, score, scope state, effective reason for rejection, and source URL.

If a `SKIP` was caused by a confirmed semantic blocker, the digest reports the blocker reason rather than only the general scope explanation.

This workflow provides a manual false-negative check without coupling quality assurance to the daily production path.

## Workflow Boundaries
The project intentionally uses several smaller workflows instead of a single large automation.
This provides several operational advantages:
- source integrations can change independently;
- failures are easier to isolate;
- processing stages can be tested separately;
- evaluation logic can be versioned independently from discovery;
- delivery behavior can evolve without modifying collection logic;
- PostgreSQL preserves state between workflow executions.

The result is a system in which workflows remain focused on orchestration and processing while persistent state provides the contract between them.
