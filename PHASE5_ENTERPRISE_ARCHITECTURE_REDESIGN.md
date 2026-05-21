# PHASE 5 — Enterprise Architecture Redesign & Future-State Platform
## Future-State Enterprise Contract Intelligence Platform
**Date:** May 19, 2026  
**Role:** Principal AI Architect / Enterprise Legal AI Architect / Principal Databricks Solution Architect  
**Scope:** Ideal future-state architecture for trusted, scalable, enterprise-grade provider contract intelligence on Databricks

---

## Executive Summary

The current Contract_Codification_Pipeline proves that large-scale provider contract extraction is feasible, but it is still fundamentally an **extraction pipeline**. The future-state platform must become a **governed Contract Intelligence Platform**: a system that combines deterministic document control, table-aware parsing, clause-level legal retrieval, amendment-aware lineage, grounded answer generation, and explicit trust controls.

The ideal architecture is **not pure RAG**, **not pure SQL**, and **not LLM-first**. It is a **hybrid legal intelligence architecture** with four coordinated planes:

1. **Document Intelligence Plane** — ingest, OCR, layout recovery, clause segmentation, table extraction, and canonical contract modeling
2. **Trusted Data Plane** — deterministic contract graph, amendment lineage, current-state materialization, Delta history, and governed business views
3. **Retrieval & Reasoning Plane** — hybrid SQL + metadata filters + vector retrieval + grounded synthesis with citations
4. **Trust, Governance & Operations Plane** — evaluation, lineage, review workflows, observability, access control, reproducibility, and compliance controls

The enterprise target is a platform that can answer three distinct classes of questions correctly:

* **Structured questions** using deterministic SQL over validated Delta tables
* **Semantic legal questions** using clause-level retrieval and grounded synthesis
* **Historical/version questions** using amendment lineage and time-aware contract state reconstruction

The most important design principle: **AI should extract, classify, compare, and summarize — but deterministic systems must govern identity, versioning, supersession, traceability, policy enforcement, and final current-state materialization.**

---

## Final Enterprise Architecture Verdict

**Verdict:** The best possible platform for this domain is a **hybrid Contract Intelligence Platform on Databricks** with:

* layout-aware OCR and table recovery
* deterministic contract/amendment graph
* clause-level legal knowledge corpus
* dual-path retrieval (SQL for structured, vector for semantic)
* citation-grounded answer synthesis
* human review for high-risk legal/financial fields
* full governance, lineage, observability, and reproducibility

**What remains AI-driven:** semantic extraction, clause classification, cross-clause comparison, answer synthesis, legal summarization, semantic retrieval.

**What becomes deterministic:** provider identity, document registration, amendment ordering rules, supersession state, table versioning, confidence policy, review state, lineage, masking, freshness checks, production serving logic.

**Architecture verdict:** The platform is highly viable, but only if redesigned from a notebook-centric extraction flow into a **Lakehouse-native, governed, multi-layer legal intelligence platform**.

---

## 1. Future-State Architecture

### 1.1 Future-State Reference Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     ENTERPRISE CONTRACT INTELLIGENCE PLATFORM             │
├────────────────────────────────────────────────────────────────────────────┤
│  EXPERIENCE LAYER                                                         │
│  * BI dashboards   * Databricks Genie   * Analyst SQL   * Legal QA UI     │
│  * API/Apps        * Review console     * Audit explorer                    │
├────────────────────────────────────────────────────────────────────────────┤
│  QUERY & REASONING LAYER                                                  │
│  * Query router (structured vs semantic vs hybrid)                        │
│  * SQL execution path                                                     │
│  * Hybrid retrieval path (metadata + vector + rerank)                     │
│  * Citation-grounded answer synthesis                                     │
├────────────────────────────────────────────────────────────────────────────┤
│  TRUSTED DATA PRODUCTS                                                    │
│  * Current contract state marts                                           │
│  * Historical contract marts                                              │
│  * Clause intelligence marts                                              │
│  * Rate intelligence marts                                                │
│  * Review, confidence, and audit marts                                    │
├────────────────────────────────────────────────────────────────────────────┤
│  CANONICAL CONTRACT INTELLIGENCE MODEL                                    │
│  * Contract entity model                                                  │
│  * Amendment graph                                                        │
│  * Clause objects                                                         │
│  * Rate schedule objects                                                  │
│  * Source spans, citations, provenance, confidence                        │
├────────────────────────────────────────────────────────────────────────────┤
│  DOCUMENT INTELLIGENCE PIPELINE                                           │
│  * Ingestion & registration                                               │
│  * OCR + layout analysis                                                  │
│  * Page, section, clause, table segmentation                              │
│  * LLM extraction + validators                                            │
│  * Human review gates                                                     │
├────────────────────────────────────────────────────────────────────────────┤
│  GOVERNANCE & OPERATIONS                                                  │
│  * Unity Catalog * lineage * masking * policies * monitoring * evaluation │
│  * run registry * model registry * prompt registry * review audit logs    │
└────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Core Design Principles

1. **Document-first, not row-first** — preserve the contract as a legal artifact before transforming to tables
2. **Structure before semantics** — recover tables, pages, sections, clauses, and spans before asking LLMs to interpret
3. **Deterministic state, probabilistic interpretation** — use AI for understanding, not for system-of-record state transitions
4. **Every answer must be grounded** — all semantic responses cite clause spans, page numbers, and source documents
5. **Every important field has provenance** — extracted, inferred, normalized, calculated, or human-approved
6. **Current state is materialized deterministically** — no user-facing “current contract” state should depend on live LLM synthesis
7. **Legal trust requires reviewability** — high-risk outputs cannot bypass human review

---

## 2. What Should Remain, Be Redesigned, Added, Removed

### 2.1 What Should Remain from the Current System

| Keep | Why |
|------|-----|
| Provider-volume discovery logic | Good deterministic foundation for large PDF corpora |
| Filename metadata parsing | Valuable first-pass signal for provider_id, document type, version |
| Direct REST invocation pattern | Better control and reliability than Spark SQL `ai_query()` for long documents |
| Adaptive concurrency | Correct for rate-limited model serving |
| Amendment-aware dedup concepts | Strong basis for current-state rate materialization |
| Base / Genie / Serving layering concept | Good abstraction; should evolve, not be discarded |
| Validation mindset (date/rate verification, audits) | Essential, should become much more rigorous |
| Delta-based serving outputs | Correct storage model for enterprise analytics |

### 2.2 What Must Be Redesigned

| Redesign Area | Why |
|--------------|-----|
| OCR | Current text linearization loses table structure |
| Chunk merge logic | Naive merge causes duplicate/misaligned extractions |
| Clause extraction model | Needs clause spans, section hierarchy, and completeness signals |
| Current canonical schema | Too extraction-centric, not contract-entity-centric |
| Audit architecture | Same-model self-audit is insufficient |
| Confidence model | Needs field-level trust and review state |
| Metadata model | Needs run, prompt, model, document, page, and extraction provenance |
| Historical/version strategy | Must support contract state reconstruction at any time |
| Production orchestration | Must move from notebook-run to job/pipeline-governed operation |

### 2.3 What Must Be Added

| Add | Why |
|-----|-----|
| Layout-aware OCR / document structure extraction | Required for fee schedules and cross-page tables |
| Page-level source spans | Required for legal traceability |
| Canonical clause registry | Required for semantic retrieval and clause intelligence |
| Amendment lineage graph | Required for historical contract state and supersession |
| Vector search over clause units | Required for semantic contract questions |
| Hybrid query router | Structured and semantic workloads require different retrieval paths |
| Human review workbench | Required for enterprise trust |
| Prompt/model registry | Required for reproducibility and governance |
| Run-level metadata tables | Required for auditability and monitoring |
| Evaluation harness | Required to measure extraction and answer quality continuously |

### 2.4 What Should Be Removed

| Remove | Why |
|--------|-----|
| Over-reliance on flat OCR text | Causes the largest extraction risk |
| Blind chunk-level extraction against full schema | Produces hallucinated metadata and redundant output |
| First-write-wins scalar merges | Silently corrupts better later chunk data |
| Same-model extraction + audit pattern | Circular validation is weak |
| Direct promotion of AI outputs to production tables | Unsafe for high-risk legal/financial fields |
| Workspace-file-only state dependency | Fragile and not enterprise durable |

---

## 3. Detailed Component Design

## 3.1 OCR Architecture

### Recommended OCR Architecture

Use a **tiered document intelligence stack**:

1. **Digital text extraction tier**
   * PDF native text extraction for text-born documents
   * Preserve coordinates, reading order, page number, font cues where available

2. **Layout analysis tier**
   * Identify text blocks, headings, tables, footers, headers, signatures, exhibits
   * Detect cross-page table continuation and repeated headers

3. **Vision/document OCR tier**
   * Apply OCR only where digital extraction is low quality or missing
   * Capture bounding boxes and page coordinates

4. **Table extraction tier**
   * Extract fee schedules as structured tables with rows, columns, headers, page links
   * Preserve row lineage back to cell coordinates and source page images

### Architecture Decision

For provider contracts, **OCR is not a text problem; it is a document structure problem.**
The redesign should model:

* page
* block
* section
* table
* row
* cell
* clause span

before LLM interpretation begins.

### Output Objects from OCR Stage

| Object | Purpose |
|--------|---------|
| `document_pages` | one row per page with OCR quality metrics |
| `document_blocks` | ordered text/layout blocks |
| `document_tables_raw` | detected tables with page ranges |
| `document_table_cells` | cell coordinates and text |
| `document_sections_raw` | heading hierarchy |
| `document_spans` | character or token spans with page anchoring |

---

## 3.2 Parsing Architecture

### Parsing Should Be Multi-Pass

**Pass 1: Deterministic document parsing**
* file registration
* filename parsing
* page count
* OCR quality scoring
* document type classification seed
* exhibit detection
* amendment indicator detection

**Pass 2: Structural parsing**
* heading tree extraction
* section boundary detection
* clause boundary detection
* rate table detection
* signature/date block detection

**Pass 3: AI-assisted semantic extraction**
* clause type classification
* amendment impact interpretation
* normalized payment model extraction
* financial protection interpretation
* cross-reference resolution

**Pass 4: Deterministic post-processing**
* canonical entity resolution
* date normalization
* current-vs-superseded state calculation
* confidence assignment rules
* review routing rules

### Why this matters

The current design asks a single LLM pass to recover structure and semantics simultaneously. The future-state design separates:

* **where content lives**
* **what the content says**
* **what it means operationally**

---

## 3.3 Recommended Chunking Strategy

### Core Principle

Chunk by **legal structure**, not character count.

### Future-State Chunk Types

| Chunk Type | Unit | Used For |
|-----------|------|----------|
| Document header chunk | Cover page + metadata sections | contract identity, parties, effective dates |
| Section chunk | Legal section with heading context | clause extraction |
| Clause chunk | Single clause or subclause | semantic retrieval, clause classification |
| Table chunk | One table with page continuation context | rate extraction |
| Amendment chunk | One amendment section + referenced prior clause/table | supersession logic |
| Signature chunk | Signature pages / execution blocks | parties, execution dates |

### Chunking Rules

1. Never split a table row across chunks
2. Never split a clause across chunks
3. Always carry parent heading context
4. For amendments, attach referenced base/agreement metadata context
5. Add preceding/following section titles as structural context instead of large overlap windows

### Recommended Overlap Strategy

Replace raw 2,000-char overlap with **semantic overlap**:

* parent section title
* nearest preceding heading
* amendment reference metadata
* previous table header row
* previous page footer/header classification only if needed

### Merge Strategy

Do not merge chunk outputs with generic JSON deep merge. Instead use **object-specific consolidation**:

* header facts consolidated by confidence and source precedence
* clause objects merged by span identity
* table rows merged by table_id + row_key
* amendment impacts merged by target clause/table references

---

## 3.4 Clause Modeling Strategy

### Canonical Clause Unit

A clause should become a first-class object with:

| Field | Description |
|------|-------------|
| `clause_id` | stable unique ID |
| `contract_id` | contract family |
| `document_id` | source document |
| `page_start`, `page_end` | citation anchors |
| `section_path` | e.g., `Article VIII > Term and Termination > 8.2` |
| `clause_title` | clause heading |
| `clause_text` | verbatim text |
| `clause_type` | termination, confidentiality, audit_rights, etc. |
| `obligation_party` | provider, payer, mutual |
| `effective_status` | current, superseded, amended, historical |
| `superseded_by_document_id` | amendment lineage |
| `citation_span_id` | precise source span |
| `review_status` | pending, approved, rejected |
| `confidence_score` | field/object confidence |

### Why Clause-First Modeling Matters

Most semantic legal value lives at the clause level, not document level. This enables:

* semantic retrieval
* clause comparison
* amendment-to-clause supersession
* legal topic clustering
* policy coverage checks

---

## 3.5 Metadata Architecture

### Required Metadata Domains

1. **Source metadata**
   * file path, volume, checksum, last modified, ingest timestamp

2. **Document metadata**
   * provider_id, provider_name, contract family, doc type, version label, execution/effective dates

3. **Run metadata**
   * pipeline_run_id, user/service principal, code version, config snapshot, start/end times

4. **Model metadata**
   * model name, version, endpoint, prompt version, temperature, token usage

5. **Extraction metadata**
   * extractor type, chunk_id, source spans, confidence, validation outcomes

6. **Review metadata**
   * reviewer, disposition, timestamp, comments, override reason

7. **Governance metadata**
   * classification, sensitivity, legal hold status, retention policy, ownership

### Recommended Control Tables

* `ctl_pipeline_runs`
* `ctl_pipeline_run_steps`
* `ctl_model_registry_snapshot`
* `ctl_prompt_versions`
* `ctl_document_registry`
* `ctl_review_queue`
* `ctl_review_decisions`
* `ctl_evaluation_runs`
* `ctl_data_quality_results`

---

## 4. Canonical Contract Model

### 4.1 The Canonical Entity Model

The platform should center on a **contract family model** rather than individual extraction tables.

### Core Entities

| Entity | Meaning |
|--------|---------|
| `provider_entity` | canonical provider/facility |
| `contract_family` | all documents governing one provider agreement |
| `contract_document` | one PDF or legal artifact |
| `contract_section` | structural section |
| `contract_clause` | clause/subclause unit |
| `rate_schedule` | one schedule/exhibit/table |
| `rate_line_item` | one payable service/program row |
| `financial_protection` | stop-loss/outlier constructs |
| `party_role` | signatory / payer / provider role |
| `amendment_event` | legal change event |
| `citation_span` | page/offset/bounding box |
| `review_event` | human review/approval |

### 4.2 Materialized Contract States

Maintain three deterministic states:

| State | Meaning |
|------|---------|
| `document_state` | what each document says |
| `historical_contract_state` | what the contract looked like after each amendment |
| `current_contract_state` | latest valid state after all supersession rules |

This allows the platform to answer:

* What is current now?
* What was current on a given date?
* Which amendment changed this rate/clause?

---

## 5. Historical / Version / Amendment Lineage Strategy

### 5.1 Amendment Lineage Strategy

Build a deterministic **amendment graph**:

* nodes = documents, clauses, schedules, line items
* edges = amends, restates, supersedes, references, terminates, clarifies

### 5.2 Contract State Reconstruction

For each contract family:

1. Register all documents with effective/execution dates
2. Determine legal ordering using deterministic precedence rules
3. Link amendment targets to specific clauses or schedules where possible
4. Materialize contract snapshots by effective date
5. Publish current state only after deterministic supersession engine runs

### 5.3 Supersession Rules

Supersession should be deterministic and policy-driven:

* explicit amendment references override inferred chronology
* later effective date overrides earlier only within same governed object domain
* restated schedules replace prior schedule versions but preserve historical lineage
* ambiguous supersession enters review queue, never auto-promotes

### 5.4 Why This Is Critical

The current system reasons mostly at document level. The future-state platform must support supersession at:

* contract level
* schedule level
* rate line level
* clause level

---

## 6. Retrieval Architecture

## 6.1 Recommended Retrieval Architecture

The ideal platform uses a **three-lane retrieval model**:

### Lane A — Deterministic Structured Retrieval
For quantitative and historical questions:

* SQL against current-state and historical-state Delta marts
* Filters by provider, date, program, network, contract family, rate category

### Lane B — Semantic Clause Retrieval
For legal language and concept exploration:

* metadata-filtered vector search over clause-level corpus
* optional section-level fallback
* rerank by legal relevance + citation density + currentness

### Lane C — Hybrid Retrieval
For mixed questions:

Example: “What is Garfield’s ICU rate, and what clause governs stop-loss escalation?”

* SQL retrieves the rate
* vector retrieval retrieves relevant stop-loss / escalation clauses
* answer synthesizer combines both with citations

### Query Router Responsibilities

The router classifies intent into:

* structured
* semantic
* historical lineage
* mixed/hybrid
* unsupported / needs human review

This router should be deterministic + rule-assisted first, optionally LLM-assisted only for ambiguous questions.

---

## 6.2 RAG Architecture

### Recommended RAG Pattern

This domain requires **grounded, citation-first RAG**, not conversational freeform RAG.

### RAG Flow

1. Query classification
2. Metadata extraction (provider, document type, clause topic, time scope)
3. Candidate retrieval:
   * SQL facts
   * vector clause candidates
   * amendment/document candidates
4. Reranking
5. Answer plan generation
6. Grounded synthesis constrained to retrieved evidence only
7. Citation formatting and trust label assignment

### Answer Guardrails

The synthesizer must:

* refuse unsupported claims
* distinguish current vs historical evidence
* distinguish exact quote vs summary
* state when retrieved evidence is incomplete or conflicting
* attach citations to every legal assertion

---

## 6.3 Embedding Strategy

### Recommended Embedding Units

Use embeddings at multiple granularities:

| Unit | Purpose |
|------|---------|
| clause | primary semantic retrieval unit |
| definition | high-value exact term retrieval |
| section | broader context fallback |
| amendment_summary | amendment discovery / change search |
| schedule caption/header | linking table/schedule references |

### Embedding Payload Design

Each embedding unit should include rich metadata:

* provider_id
* canonical_provider_name
* contract_family_id
* document_id
* document_type
* effective_date
* current_flag
* clause_type
* section_path
* citation anchors
* review_status
* confidence tier

### Recommendation

Do **not** embed whole documents as the primary retrieval unit. For this corpus, clause/definition/section-level indexing is superior.

---

## 6.4 Vector Indexing Strategy

### Recommended Vector Indices

Create separate indices by semantic role:

| Index | Contents | Why |
|------|----------|-----|
| `contract_clause_index` | clauses + subclauses | primary legal retrieval |
| `contract_definition_index` | defined terms | exact concept grounding |
| `contract_section_index` | section summaries + titles | context expansion |
| `contract_amendment_index` | amendment summaries / impacts | change discovery |

### Index Design Principles

* keep clause and definition retrieval separate
* include current/historical metadata filters
* support provider-scoped and corpus-wide retrieval
* store exact citation references alongside vectors

---

## 6.5 Re-ranking Strategy

For this domain, reranking should be **lightweight but domain-aware**.

### Recommended Rerank Features

* semantic similarity score
* provider/entity match score
* currentness score
* clause type match
* citation completeness score
* review approval bonus
* confidence tier bonus

### Recommendation

Use deterministic feature-based reranking first. Only introduce model-based reranking if retrieval quality remains insufficient after clause-level indexing and metadata filtering.

For a 47K–100K clause-scale corpus, heavy cross-encoder reranking is optional, not mandatory.

---

## 7. Recommended Metadata Model

### 7.1 Trust-Aware Field Provenance Model

Every field should have:

| Field Suffix | Meaning |
|-------------|---------|
| `_value` | actual value |
| `_provenance_type` | extracted / inferred / normalized / calculated / human_override |
| `_confidence_score` | 0–1 score |
| `_confidence_tier` | high / medium / low |
| `_citation_span_id` | exact source anchor |
| `_validation_status` | unverified / auto_verified / failed / human_verified |
| `_review_status` | not_required / pending / approved / rejected |

### 7.2 Required High-Value Metadata for Rates

For every `rate_line_item`, include:

* service_line_raw
* service_line_normalized
* program_raw
* program_normalized
* network_raw
* network_normalized
* rate_text
* rate_numeric
* rate_basis
* source_table_id
* source_row_id
* page_number
* bounding_box_ref
* extracted_by_model
* reviewed_by_human

### 7.3 Clause Metadata Model

For every clause:

* legal topic
* clause type
* obligation direction
* remedy / penalty indicator
* notice period indicator
* amendment impact status
* currentness
* citation completeness
* quoted_text_hash

---

## 8. Recommended Trust Architecture

### 8.1 Trust Stack

```
Level 1: Source Preservation
Level 2: Structural Recovery
Level 3: Field Extraction Provenance
Level 4: Deterministic Validation
Level 5: Human Review for High-Risk Items
Level 6: Grounded Retrieval & Citation
Level 7: Policy-Enforced Production Serving
```

### 8.2 Trust Tiers for Outputs

| Tier | Output Type | Allowed Use |
|------|-------------|-------------|
| T0 | Raw OCR / unreviewed extraction | analyst exploration only |
| T1 | Auto-validated structured data | internal analytics with caveats |
| T2 | Human-reviewed financial/legal critical fields | operational decision support |
| T3 | Citation-grounded current-state outputs | enterprise legal intelligence consumption |

### 8.3 What Must Become Deterministic

| Deterministic Domain | Why |
|----------------------|-----|
| provider identity resolution | system-of-record join key |
| document registration | chain of custody |
| amendment ordering policy | legal version control |
| supersession materialization | current-state correctness |
| review workflow state | governance |
| masking and access policy | compliance |
| freshness and SLA checks | production reliability |
| current-state serving tables | consistent downstream analytics |

### 8.4 What Should Remain AI-Driven

| AI-Driven Domain | Why |
|------------------|-----|
| clause topic classification | semantic task |
| amendment impact interpretation | legal language understanding |
| clause comparison | semantic similarity task |
| legal summarization | natural language compression |
| semantic retrieval expansion | concept-based search |
| anomaly explanation | narrative explanation of detected drift |

---

## 9. Recommended Governance Framework

### 9.1 Governance Layers

1. **Data governance**
   * Unity Catalog tables, schemas, comments, ownership, lineage

2. **Security governance**
   * sensitivity tagging, column masks, access control, service principals

3. **AI governance**
   * prompt versioning, model versioning, evaluation logs, approval gates

4. **Legal governance**
   * review status, citation sufficiency, currentness certification

5. **Operational governance**
   * SLAs, alerts, rerun controls, release management

### 9.2 Enterprise Security Design

* Unity Catalog for all managed tables
* column masks for tax_id, npi, legal-sensitive identifiers
* row filtering where business segregation is needed
* service principal–driven scheduled execution
* secrets in Databricks secrets scope / managed auth
* audit logs for access to sensitive legal content
* environment separation: dev / test / prod catalogs

### 9.3 Recommended Governance Tables

* `gov_data_asset_inventory`
* `gov_column_sensitivity_tags`
* `gov_model_prompt_lineage`
* `gov_review_policy_rules`
* `gov_release_certifications`
* `gov_contract_data_exceptions`

---

## 10. Recommended Evaluation Framework

### 10.1 Evaluation Must Cover Four Distinct Layers

| Layer | What to Evaluate |
|------|------------------|
| OCR / structure | page text fidelity, table detection recall, clause boundary accuracy |
| extraction | field precision/recall, clause completeness, table row accuracy |
| retrieval | top-k relevance, citation recall, currentness correctness |
| answering | groundedness, legal faithfulness, unsupported claim rate |

### 10.2 Gold Dataset Strategy

Create a curated benchmark of:

* representative base agreements
* multi-amendment chains
* long fee schedules
* cross-page tables
* settlement documents
* high-risk legal clauses

### 10.3 Metrics

#### Extraction Metrics
* rate row precision / recall
* service-rate attribution accuracy
* date extraction precision
* clause completeness score
* amendment linkage accuracy

#### Retrieval Metrics
* clause retrieval recall@k
* citation completeness@k
* current-vs-historical retrieval accuracy
* provider filter precision

#### QA Metrics
* grounded answer rate
* unsupported claim rate
* citation correctness rate
* abstention correctness rate

### 10.4 Evaluation Cadence

* pre-deployment benchmark gate
* every model/prompt change
* weekly drift regression suite
* monthly human-reviewed audit sample

---

## 11. Recommended Human Review Workflow

### 11.1 Review Queue Design

Route items to review based on policy rules.

### Mandatory Review Triggers

* settlement-related extractions
* stop-loss and outlier thresholds
* rate changes above policy threshold
* ambiguous amendment supersession
* low-confidence effective dates
* low-confidence clause completeness
* new provider onboarding
* extraction conflicts across documents

### 11.2 Review States

| State | Meaning |
|------|---------|
| `pending_review` | waiting for human decision |
| `approved` | human accepted |
| `corrected` | human edited value |
| `rejected` | extraction invalid |
| `escalated_legal` | requires attorney/legal ops review |

### 11.3 Review Console Capabilities

The review application should show side-by-side:

* source PDF page image
* OCR text with highlights
* extracted field or clause
* citation span
* prior contract/amendment value
* current-state impact if approved

This is essential for trust and throughput.

---

## 12. Monitoring & Observability

### 12.1 Production Monitoring Requirements

Monitor:

* ingest volume changes
* OCR quality distribution
* extraction success/failure rate
* table row deltas by provider and document type
* duplicate rate spikes
* NULL date spikes
* low-confidence output spikes
* retrieval latency and hit quality
* unsupported-answer rate
* review backlog size
* stale provider count

### 12.2 Operational Telemetry Tables

* `ops_pipeline_events`
* `ops_document_processing_metrics`
* `ops_extraction_metrics_daily`
* `ops_retrieval_metrics`
* `ops_answer_quality_metrics`
* `ops_review_backlog_metrics`
* `ops_data_freshness_status`

### 12.3 Alerting

Alert on:

* zero-file runs
* unusually low clause counts
* >20% shift in rate row counts
* sudden rise in NULL effective dates
* vector retrieval failure
* review SLA breaches
* stale current-state data

---

## 13. Scalability Architecture

### 13.1 Processing Architecture

For 10,372+ contracts with ongoing increments:

* use distributed preprocessing for document registration and OCR scoring
* parallelize LLM extraction at document/section/table level with backpressure control
* persist intermediate structure objects in Delta, not only JSON files
* process incrementally by changed/new documents
* support selective reprocessing by provider, document family, or extraction version

### 13.2 Recommended Workload Split

| Workload | Best Execution Mode |
|---------|---------------------|
| ingestion, OCR scoring, structure assembly | Spark batch / serverless jobs |
| LLM extraction orchestration | Python notebook or Python file task with async concurrency |
| vector index refresh | scheduled batch refresh |
| serving mart builds | SQL or Spark jobs |
| evaluation suites | scheduled batch jobs |
| review queue materialization | SQL jobs |

### 13.3 Scalability Principle

Do not re-run the entire corpus for every change. Support:

* append-only document registry
* versioned extraction outputs
* selective backfills
* parallel re-embedding only for changed clause objects

---

## 14. Recommended Technology Stack

### 14.1 Databricks-Native Services

| Need | Recommended Databricks Service |
|------|-------------------------------|
| governed storage and access | Unity Catalog |
| raw + refined + serving tables | Delta Lake |
| orchestration | Lakeflow Jobs |
| SQL serving and BI access | Databricks SQL |
| vector retrieval | Databricks Vector Search |
| semantic QA/chat | Genie + custom app/API |
| model endpoint invocation | Model Serving / external model endpoints |
| monitoring and logs | system tables + Delta ops tables + jobs monitoring |
| notebook/engineering development | Databricks notebooks / repos / files |
| review application | Databricks Apps or external lightweight review UI |

### 14.2 Recommended Databricks Services by Plane

**Ingestion / Processing Plane**
* Lakeflow Jobs
* Delta tables
* Unity Catalog volumes or governed file storage

**Retrieval Plane**
* Databricks Vector Search
* Databricks SQL
* Genie space over curated current-state and historical tables

**Governance Plane**
* Unity Catalog lineage, masks, tags, comments
* MLflow / model lineage where applicable
* system tables for operational telemetry

---

## 15. Serving / Query Architecture

### 15.1 Serving Model

Publish four major product layers:

| Layer | Purpose |
|------|---------|
| `curated_current` | trusted current-state contract facts |
| `curated_historical` | historical states and amendment timeline |
| `curated_legal_corpus` | clauses, definitions, sections, citations |
| `curated_review_audit` | confidence, review, quality, and exceptions |

### 15.2 Query Access Patterns

* executives and analysts use SQL dashboards and serving views
* legal operations uses review UI and citation explorer
* AI assistants use hybrid retrieval APIs backed by Delta + Vector Search
* auditors use lineage + run history + review history tables

---

## 16. Recommended Migration Strategy

### Phase A — Stabilize Current Pipeline

* add model version, prompt version, extraction timestamp
* add run log tables
* extend Delta retention
* move JSON state to governed durable storage
* add PII masking and sensitivity tagging

### Phase B — Introduce Structural Layer

* implement page/block/table/clause intermediate Delta tables
* add page citations and span IDs
* replace naive chunking/merge with structure-aware extraction

### Phase C — Build Canonical Contract Model

* create contract_family, contract_document, contract_clause, rate_line_item entities
* implement deterministic amendment graph and supersession engine
* materialize historical and current states

### Phase D — Build Hybrid Retrieval

* create clause and definition embeddings
* deploy Vector Search indices
* build query router and grounded synthesis service
* expose legal citation answers and hybrid analytics answers

### Phase E — Governance & Review Hardening

* deploy review queue and approval workflows
* add evaluation benchmark suite
* add production monitoring, alerts, and release certification

---

## 17. Quick Wins

1. Record `model_name`, `model_version`, `prompt_version`, `extraction_timestamp` immediately
2. Add `provenance_type` fields for key columns: program, network, effective_date, rate_numeric
3. Apply column masks to `tax_id` and `npi`
4. Create `ctl_pipeline_runs` and `ctl_review_queue`
5. Add page number capture to all extracted clauses and rates
6. Separate current-state vs historical-state marts more explicitly
7. Create a Genie space on trusted serving + clause tables

---

## 18. High-Impact Improvements

1. Replace flat OCR with layout/table-aware parsing
2. Implement deterministic amendment supersession engine
3. Move from document-level to clause/rate-line-level canonical entities
4. Add hybrid SQL + vector retrieval
5. Build citation-grounded answer generation with abstention behavior
6. Introduce policy-driven human review for high-risk fields

---

## 19. Long-Term Strategic Architecture

The long-term target is a **contract operating system** for provider agreements:

* every contract becomes a queryable legal object model
* every amendment becomes a graph event
* every answer is grounded in citations
* every current-state fact is deterministically materialized
* every high-risk field is reviewable and auditable
* every semantic legal question is answered through clause retrieval, not raw document prompting

This architecture can support future use cases beyond extraction:

* reimbursement strategy analysis
* negotiation intelligence
* clause policy compliance scanning
* amendment impact simulation
* enterprise legal knowledge retrieval
* payer-provider portfolio benchmarking

---

## 20. Productionization Roadmap

### 0–30 Days
* trust metadata capture
* run logs
* PII masking
* retention hardening
* schedule pipeline execution

### 30–90 Days
* page-level citation capture
* structural parsing layer
* review queue MVP
* baseline evaluation suite
* clause corpus cleanup

### 90–180 Days
* canonical contract model
* amendment graph
* current/historical deterministic marts
* vector search deployment
* hybrid retrieval service

### 180–360 Days
* full legal QA experience
* approval-gated publishing
* continuous regression testing
* contract intelligence apps and domain APIs

---

## Recommended Databricks Services Summary

* **Unity Catalog** for governance, masking, lineage, tags, access control
* **Delta Lake** for canonical entities, historical states, serving marts, and telemetry
* **Lakeflow Jobs** for orchestration and scheduled production runs
* **Databricks SQL** for structured analytics and serving queries
* **Databricks Vector Search** for clause/definition semantic retrieval
* **Genie** for trusted natural-language access over curated structured data
* **Databricks Apps** for review workflows and legal analyst experiences
* **Model Serving / external endpoints** for LLM extraction and synthesis

---

## Final Enterprise Architecture Verdict

The best possible future-state platform is a **hybrid, governed, clause-aware Contract Intelligence Platform** built natively on Databricks. It preserves the current system’s strongest assets — scalable extraction, layered Delta serving, and amendment-aware thinking — while replacing its weakest foundations with structure-aware parsing, canonical contract modeling, deterministic lineage, hybrid retrieval, and policy-driven trust controls.

**This should not evolve into a generic RAG chatbot.** It should evolve into a **trusted enterprise contract intelligence system** where:

* structured facts come from deterministic current-state Delta marts
* semantic legal understanding comes from clause-level retrieval
* amendment history is represented as a governed graph
* every answer is cited, explainable, and trust-labeled
* high-risk outputs require review before enterprise reliance

That is the architecture most likely to succeed for provider contracts, legal traceability, amendment-heavy documents, and enterprise-scale Databricks operations.
