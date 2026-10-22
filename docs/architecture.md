# System Architecture

This document describes the internal architecture of the AI-Powered Freelance Opportunity Aggregator in more detail than the main README.

The system is designed as a set of independent n8n workflows connected through PostgreSQL rather than as one large monolithic automation.

![System Architecture](../assets/diagrams/system-architecture.svg)

## Architectural Goals

The architecture was designed around several practical requirements:

- isolate source-specific collection logic from shared processing logic;
- keep PostgreSQL as the persistent source of truth between workflow executions;
- avoid passing large amounts of discovery data through one long in-memory workflow chain;
- perform deterministic filtering before more expensive semantic evaluation;
- keep the LLM responsible for interpretation rather than final decision-making;
- make repeated executions safe through deduplication and persisted processing state;
- allow individual layers to be modified or reprocessed without rebuilding the entire pipeline.

These requirements led to a layered architecture composed of 14 coordinated workflows:

- 8 source-specific discovery workflows;
- 1 Daily Orchestrator;
- 1 Hard Filter workflow;
- 1 AI Evaluation / Semantic Filter workflow;
- 1 Telegram Daily Digest workflow;
- 1 DB Housekeeping workflow;
- 1 SKIP Audit workflow.

The daily production pipeline and the operational support workflows are intentionally separated. Discovery, filtering, semantic evaluation, and delivery belong to the main daily path, while database maintenance and decision-quality auditing run independently on their own schedules.

## Architectural Layers

### 1. Discovery Layer

The Discovery Layer contains independent workflows responsible for collecting opportunities from heterogeneous external sources.

Different sources may expose data through different mechanisms such as structured APIs, feeds, structured web data, or email notifications.

Each discovery workflow is responsible for:

1. retrieving source data;
2. extracting the relevant opportunity fields;
3. normalizing the source-specific representation;
4. producing a shared internal opportunity structure;
5. upserting the resulting record into PostgreSQL.

This boundary is important because downstream workflows do not depend on the original ingestion mechanism.

Once an opportunity has been normalized and persisted, the rest of the system operates against the database representation rather than against source-specific payloads.

### 2. Daily Orchestrator

The Daily Orchestrator coordinates the production execution sequence.

It triggers the eight discovery workflows and waits for them to complete before starting shared downstream processing.

Each discovery call is configured as an independent failure boundary. If one source-specific workflow fails, the error is surfaced as a regular completion result rather than terminating the parent orchestration run. The remaining discovery workflows can finish normally, the synchronization stage can complete, and downstream processing continues against the data successfully persisted by the available sources.

This behavior keeps partial source outages from disabling the entire daily pipeline while preserving source-level observability for troubleshooting.

The discovery branches converge through a Merge node, but the Merge node is primarily used as a synchronization barrier rather than as the main data transport mechanism.

Discovery workflows already persist their results directly to PostgreSQL.

This means downstream processing does not depend on carrying every discovered opportunity through one large n8n execution context.

Conceptually, the orchestrator coordinates:

```text
Schedule
    ↓
Discovery Workflows
    ↓
Synchronization
    ↓
Hard Filter
    ↓
AI Evaluation
    ↓
Telegram Delivery
```

The orchestration layer therefore controls execution order while PostgreSQL carries the persistent system state.

## PostgreSQL as the State Layer

PostgreSQL acts as the shared persistent state layer between otherwise independent workflows.
The production data model is organized around three persistent areas:
```
opportunities
    │
    ├── opportunity_ai_evaluations
    │
    └── opportunity_delivery_log
```

### Opportunities

The opportunity store contains normalized opportunity data together with discovery and deterministic filtering state.
Hard-filter results are stored alongside the opportunity so that filtering does not need to be recomputed unnecessarily on every downstream execution.

### AI Evaluations

Semantic evaluation is stored separately from the opportunity itself.
This separation allows evaluation logic to be versioned independently from collected data.
A single opportunity can therefore be re-evaluated using a newer evaluation version without silently overwriting the historical result.

Each evaluation also stores provenance metadata alongside the structured semantic result. The persisted record includes the evaluation version, model name, prompt version, raw structured model response, and evaluation timestamp.

This separates the model's original interpretation from the deterministic Decision Engine output and makes later comparison, debugging, and re-evaluation easier.

The semantic evaluation layer also has an explicit failure path. If model execution or structured-output parsing fails, the record is routed through a fallback transformation that produces a REVIEW decision with low-confidence scope state and explicit uncertainty metadata. Failed or malformed AI output therefore cannot silently become a SKIP.

### Delivery Log

Delivery state is stored separately from both opportunities and evaluations.
A successful delivery records the evaluation that was sent and the delivery channel.
Future delivery queries can therefore exclude already delivered evaluations while retaining a history of previous output.

## Deterministic and Probabilistic Boundaries

One of the main architectural decisions was to separate deterministic processing from probabilistic semantic interpretation.

The pipeline follows this sequence:
```
Deterministic preprocessing
        ↓
Deterministic Hard Filter
        ↓
Structured LLM Evaluation
        ↓
Deterministic Decision Engine
        ↓
Deterministic Delivery Selection
```

The LLM is intentionally placed inside a constrained semantic evaluation layer rather than being allowed to control the entire workflow.
Its responsibility is to interpret information that is difficult to evaluate reliably with simple rules, such as:
- relevance of the actual task;
- relationship between required and demonstrated skills;
- manageability of skill gaps;
- role and delivery fit;
- semantic blockers;
- meaningful uncertainty.
The model returns structured output.

A separate deterministic Decision Engine then converts that structured evaluation into one of three decision-support states:
```
APPLY
REVIEW
SKIP
```
This makes the final classification predictable for the same structured input and prevents free-form model reasoning from directly triggering downstream actions.

## Reprocessing and Versioning

Processing state is persisted so that workflows can select only records that require work.
For example, semantic evaluation can query for opportunities that:
- passed the current hard-filter version;
- do not yet have an evaluation for the current evaluation version.

This creates a simple reprocessing mechanism.

Changing an evaluation version does not require deleting existing opportunity data or rebuilding the discovery layer. The evaluation workflow can process only the records missing the new version.

The same principle applies to delivery state: already delivered evaluations are excluded through persisted delivery records rather than through temporary workflow memory.

## Failure Boundaries

The system uses different failure policies depending on the responsibility of each layer.

Source-specific discovery workflows are isolated at the Daily Orchestrator boundary. A failure in one discovery source does not terminate the remaining source branches or prevent downstream processing from continuing with data already persisted by successful workflows.

Transient failures in network-dependent discovery and database operations use bounded retry behavior where appropriate.

The AI Evaluation workflow uses a separate error path for malformed or failed semantic evaluations. Instead of silently dropping the opportunity or classifying it as SKIP, the failure is transformed into an explicit REVIEW result with uncertainty metadata and can be persisted through the normal evaluation storage path.

Operational support workflows are also isolated from the daily production pipeline. DB Housekeeping and SKIP Audit run independently, so a maintenance or quality-control failure cannot interrupt daily opportunity discovery, filtering, evaluation, or delivery.

PostgreSQL remains the persistent recovery boundary across these workflows. Successfully completed work is retained even when another integration or workflow fails later in the process.

## Extensibility

The architecture allows several parts of the system to evolve independently.

A new discovery source can be added by implementing the same normalized persistence contract without modifying the Hard Filter or AI Evaluation logic.

Evaluation logic can be changed through a new evaluation version without rebuilding discovery workflows.
Delivery channels can also be extended independently because delivery state is stored separately from evaluation state.

The same architecture could therefore support additional outputs such as email, Slack, dashboards, or other notification channels without changing the core discovery and evaluation layers.

## Design Principle

The central architectural principle is:
Workflows coordinate processing; PostgreSQL preserves system state.

This keeps the automation modular, makes repeated execution safer, reduces coupling between integrations, and allows deterministic logic, semantic evaluation, and delivery behavior to evolve independently.

> **Workflows coordinate processing; PostgreSQL preserves system state.**

