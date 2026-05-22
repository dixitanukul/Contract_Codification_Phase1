# 12-Month Target Enterprise Architecture

*The ideal evolution of the Provider Contract Intelligence Platform into a true enterprise system of record for what is contractually in force. Synthesizes the seven prior phase audits into a forward design. Assumes the current Databricks-native footprint as the starting point.*

---

## Design principles (the spine of everything below)

1. **State, not search, is the product.** The system's job is to answer "what governs facility F, program P, service S, on date D — and prove it." Retrieval and analytics are views onto that state. (Phases 3–4.)
2. **The contract is the aggregate root**, versioned by amendment events — not the PDF, not the provider_id. (Phases 2–3.)
3. **Bitemporal by default.** Every fact carries effective time *and* knowledge time, so "what's in force" and "what did we believe, when" are both queryable. (Phase 3.)
4. **Trust is a first-class output.** Every answer ships with provenance, coverage, confidence, and an as-of date. (Phases 4, 6.)
5. **Separate the engine from the warehouse.** Databricks is the data/ML plane; reasoning and serving become governed services. (Phase 1: today everything is notebook-resident with no orchestration.)
6. **Spend-weighted.** Contract intelligence without claims/utilization volume cannot quantify what matters; integrate it. (Phase 5.)

---

## Part A — What goes where, and what's missing

### 1. What should remain in Databricks
Databricks stays the **data and ML plane** — its strengths. Keep there:
- **Ingestion & OCR** (the 2-tier pipeline) and the **document-AST extraction** — heavy, batch, GPU/compute-bound work.
- **The medallion lake**: bronze (raw OCR/AST), silver (normalized facts), and the **gold bitemporal state store** as Delta tables with Unity Catalog governance.
- **Vector indexes** (Databricks Vector Search) and **embedding/model serving** for LLM calls.
- **Batch analytics & ML**: benchmarking aggregates, clause-similarity matrices, model training/eval jobs.
- **Orchestration**: replace the implicit notebook-numbering order with **Databricks Workflows / DLT** (today there is *no* orchestration — Phase 1).

Rule of thumb: anything batch, data-gravity-bound, or compute-heavy stays in Databricks. Anything interactive, low-latency, or transactional moves out.

### 2. What should become services / APIs
Promote from notebooks to versioned, authenticated services (Databricks Apps / Model Serving / a thin API tier):
- **Contract State API** — the keystone: `state(facility, program, service, as_of_date) → governing rate_rule + clause + source_amendment + provenance`. Every UI, dashboard, and bot calls this; "current" is defined exactly once.
- **Retrieval API** — hybrid (lexical + semantic + reranker), state- and entity-filtered, returning spans with provenance.
- **Grounded QA / Reasoning API** — orchestrates retrieval + state + LLM synthesis + citation verification; returns answer + citations + confidence + coverage.
- **Negotiation/Renewal API** — scenario modeling, spend-weighted proposals, alerts.
- **Entity Resolution API** — facility/provider/contract resolution shared by every surface (today each notebook re-implements `ILIKE` matching — Phase 5).

These are stateless request/response services in front of the Databricks data plane; they give you auth, rate-limiting, versioning, SLAs, and observability that notebooks can't.

### 3. What should become modular libraries
Extract the logic currently copy-pasted across notebooks into versioned, unit-tested Python packages (one repo, semantic versioning, CI):
- **`contract-extraction`** — OCR, AST parsing, schema, prompt templates, JSON repair (today duplicated in `_config`/04/05).
- **`amendment-engine`** — event sourcing, supersession fold, bitemporal state computation.
- **`reimbursement-model`** — rate-rule objects, service-line taxonomy, condition logic (stop-loss/carve-out/lesser-of).
- **`entity-resolution`** — facility/provider/contract resolution with Tax-ID/NPI anchoring.
- **`retrieval-core`** — chunking, contextual embedding, hybrid fusion, reranking, citation verification.
- **`eval-harness`** — golden sets, metrics, regression gates.
The notebooks and services then *import* these; logic stops drifting (Phase 4 found the same SQL-gen/schema logic diverging across 19/23).

### 4. What should become knowledge-graph infrastructure
The contract domain is inherently a graph; today it's flattened into rows (Phase 3). Stand up a **contract knowledge graph** as the canonical state layer:
- **Nodes**: HealthSystem → Facility → ContractingEntity (TIN/NPI) → Contract → {Document, Exhibit, Section, Clause, RateRule, Obligation, Definition, Party}.
- **Edges**: `AMENDS`, `REPLACES_EXHIBIT`, `SUPERSEDES`, `DEFINES`, `REFERENCES`, `DELEGATES`, `PARTY_TO`.
- **Every node bitemporal** (effective_from/to, knowledge_from/to) with a provenance span.
- **Implementation**: the graph can be *materialized* from Delta (graph-as-views on Unity Catalog tables) and/or a property-graph store for traversal; the Contract State API reads it. Graph traversal (definitions used by a clause, exhibits an amendment replaced, prior versions) becomes a first-class query, not a flattened string.

### 5. Governance layers that are missing
Today: a dev schema (`dev_adb.raw`), reads from prod volumes, no lineage discipline (the full-text index is sourced from a file checkpoint, off-catalog — Phase 1). Add:
- **Unity Catalog as the governance backbone** — every table/index/model registered, with column-level lineage end-to-end (no off-catalog file sources).
- **PHI/PII data classification & access control** — provider contracts contain Tax IDs, NPIs, settlement amounts; row/column masking and purpose-based access by role.
- **Data contracts & schema enforcement** — eliminate the schema drift that breaks the text-to-SQL paths (Phase 4); versioned schemas with CI checks.
- **Promotion path dev → staging → prod** with environment isolation (not prod-read/dev-write).
- **Change management & audit log** for the state layer (who/what/when changed a governing fact).
- **Model & prompt governance** — registered, versioned prompts and models (today prompts are notebook literals with hardcoded counts — Phase 6).

### 6. Observability layers that are missing
Today: `print()` statements and a per-run `_progress.json` (Phase 1, 5). Add:
- **Pipeline observability** — job success/latency/freshness, extraction throughput, % corpus covered (the 34% gap must be a tracked, visible metric — Phase 3/6).
- **Data-quality monitoring** — null/anomaly rates on rates, dates, supersession, identity collisions; alerting on drift.
- **Retrieval/RAG telemetry** — query latency, recall proxies, abstention rate, citation-faithfulness pass rate, "no evidence" rate.
- **LLM observability** — token cost, latency, error/timeout rates, output-validity rate, hallucination flags.
- **Model/embedding drift** — distribution monitoring on embeddings and extraction outputs.
- **Usage analytics** — which questions users ask, where answers fail, feeding the eval set.

### 7. Evaluation frameworks that are missing
Today: "validation" tests routing accuracy and abstention on absurd queries; no extraction or retrieval quality measurement (Phases 4–6). Add:
- **Extraction eval** — a human-labeled gold set per document type; field-level precision/recall (rates, dates, parties, supersession), tracked over time.
- **Retrieval eval** — labeled query→relevant-span sets; recall@k, nDCG, MRR; per-index and per-query-type.
- **Grounding/faithfulness eval** — does each cited span entail its claim (NLI/LLM-judge), measured, not assumed.
- **Current-state eval** — a hand-built "what was in force on date D" answer key for a sample of providers, to measure supersession correctness (the hardest and most important).
- **End-to-end answer eval** — accuracy, abstention calibration, citation correctness on a domain question bank.
- **Regression gates in CI** — no model/prompt/schema change ships if eval drops. This is the layer that converts "demo" into "system."

### 8. Trust architecture that is missing
Today trust is simulated (confidence = cosine similarity; full-text hardcoded "CURRENT"; unverified citations — Phase 4). Build a real trust stack:
- **Span-level provenance** on every fact and answer (offset into source, not just filename).
- **Citation-faithfulness verification** before an answer is returned; drop/flag unsupported claims.
- **Calibrated confidence** decoupled into *retrieval confidence* and *answer confidence*; never derived from similarity.
- **Coverage transparency** — every answer states what fraction of relevant documents were available ("based on 4 of 6 amendments; Exhibit C unresolved").
- **As-of dating** on every state answer.
- **Human-in-the-loop** review queues for low-confidence extractions and any answer that proposes a number or a redline; reviewer feedback flows back to eval and training.
- **Honest abstention** driven by faithfulness/coverage, not a prompt that discourages it.

### 9. Legal reasoning architecture that is missing
Today there is no legal reasoning — only text retrieval and document-level "latest" heuristics (Phases 2–4). Add:
- **Supersession reasoning** — fold typed amendment operations (`REPLACE_EXHIBIT`, `MODIFY_SECTION`, `RESTATE_ALL`) over the contract chain to derive in-force state at exhibit/clause granularity, driven by the amendment's *declared* impact and supersession language (the corpus states it explicitly — 90% of amendments — and the system ignores it).
- **Defined-term resolution** — link definitions to the clauses that use them so a clause's operative meaning is computed, not guessed.
- **Clause materiality extraction** — pull the actual term values (notice days, caps, percentages) so comparison is on substance, not embedding resemblance (Phase 5).
- **Conflict & precedence reasoning** — handle "notwithstanding," "in lieu of," base-vs-amendment precedence.
- **Obligation modeling** — represent operational duties (timely filing, DOFR, delegation, notice) as dated, queryable rules.
- **Reimbursement adjudication logic** — rate + conditions assembled into a rule precise enough to price a sample claim.

### 10. Semantic state architecture that is missing
This is the missing center of gravity — the bitemporal contract state layer (Phase 3):
- **Canonical entity layer** — resolved health-system/facility/contracting-entity/contract identities anchored on Tax-ID/NPI.
- **Bitemporal fact store** — every rate, clause, obligation with effective and knowledge intervals and provenance.
- **Event-sourced amendment ledger** — amendments as immutable events; state is the deterministic fold.
- **Controlled vocabularies** — service-line taxonomy, program/network canon, code systems (CPT/HCPCS/Rev/DRG) — replacing free-text keys and fabricated defaults.
- **A single "current()" definition** computed once in the state layer and consumed everywhere (today it's computed three inconsistent ways — Phases 2,5).

---

## Part B — Target architectures

### Target system architecture
```
┌── DATA PLANE (Databricks) ─────────────────────────────────────────────┐
│  Ingest/OCR → Document AST → Silver facts → GOLD: bitemporal state +    │
│  knowledge graph (Delta + Unity Catalog) | Vector indexes | ML/eval jobs│
│  Orchestrated by Workflows/DLT. Governed by Unity Catalog lineage.      │
└───────────────┬────────────────────────────────────────────────────────┘
                │ (registered models, governed tables, graph views)
┌── SERVICE PLANE (APIs) ─────────────────────────────────────────────────┐
│  Entity Resolution · Contract State · Retrieval · Grounded QA ·          │
│  Negotiation/Renewal · all built on shared libraries, all observable     │
└───────────────┬────────────────────────────────────────────────────────┘
                │
┌── EXPERIENCE PLANE ─────────────────────────────────────────────────────┐
│  Provider 360 · Clause Library · Renewal/Obligation Calendar ·           │
│  Negotiation Workspace · Conversational QA — role-based, with provenance │
└──────────────────────────────────────────────────────────────────────────┘
   Cross-cutting: Governance · Observability · Evaluation · Trust · HITL
```
Three planes: Databricks as data/ML, a service tier for low-latency governed access, an experience tier for users — with governance/observability/eval/trust spanning all three.

### Target semantic architecture
A **bitemporal contract knowledge graph** as the single source of truth. Entities resolved on hard identifiers; amendments as events; rates and clauses as versioned, effective-dated nodes with provenance; controlled vocabularies for service lines, programs, and codes. Marts and vector indexes are **projections regenerated from the graph**, so analytics, search, and QA never diverge on "what's current."

### Target retrieval architecture
**State-aware hybrid retrieval.** Pre-filter by entity + program + service + as-of-date and `in_force`; run lexical (BM25, for codes/dollar amounts/defined terms) + semantic (contextual embeddings of structure-aware chunks carrying who/when/where headers) in parallel; fuse with RRF; rerank with a cross-encoder; return **spans with provenance**, then verify citation faithfulness before answering. Retrieval is an access path into the state layer — the state API is the first hop for "as-of" questions, retrieval supplies the supporting language.

### Target intelligence architecture
Apps reorganized around the entities teams think in — **Provider Relationship, Contract, Fee Schedule, Obligation, Negotiation Cycle** — all reading the state API and a **governed metrics/semantic layer** (one definition of current rate, peer, spend, expiration). **Spend-weighted** throughout via a claims/utilization join, so benchmarks and proposals are in annualized dollars. Negotiation and renewal become **stateful workflows** (scenario → proposal → redline → approval → monitor), not one-shot scripts. Every output carries confidence and coverage.

### Target production architecture
- **CI/CD** for libraries, prompts, models, and pipelines, with **eval regression gates** as merge blockers.
- **Environment isolation** dev → staging → prod under Unity Catalog; no prod-read/dev-write.
- **Orchestrated, monitored pipelines** (Workflows/DLT) with freshness SLAs and data-quality alerts.
- **Model/prompt registry** with versioning and rollback.
- **Full observability** (pipeline, data-quality, RAG, LLM-cost, drift) feeding dashboards and alerts.
- **HITL review queues** wired into the trust stack and back into eval.
- **Security/compliance**: PHI/PII classification, role-based + purpose-based access, audit logging.

---

## 12-month sequencing (how to get there without a rewrite)

**Quarter 1 — Foundation & integrity.** Fix the same-day defects (hardcoded `CURRENT`, schema drift, swallowed errors). Stand up Unity Catalog governance + Workflows orchestration. Extract the first shared libraries (`contract-extraction`, `entity-resolution`). Build the extraction & retrieval **eval harness** with gold sets — you cannot improve what you don't measure.

**Quarter 2 — Semantic state.** Introduce the `Contract` entity and re-key consolidation to contract_id; build the **bitemporal fact store** and **event-sourced amendment engine** with declared-impact supersession; ship the **Contract State API** (v1). Add service-line/program controlled vocabularies. Re-anchor identity on Tax-ID/NPI.

**Quarter 3 — Retrieval & trust.** Hybrid + reranked, state-aware retrieval; span-level citations + faithfulness verification; calibrated confidence and coverage on every answer; the knowledge-graph materialization and traversal. Stand up RAG/LLM observability.

**Quarter 4 — Intelligence & production.** Integrate claims/spend; rebuild negotiation/renewal as spend-weighted, stateful workflows over the state API; Provider 360 + clause library + obligation calendar as deployed, role-based apps with HITL; close the corpus-coverage gap; full CI/CD with eval gates.

Each quarter ships usable value and the believable demos from Phase 7 become trustworthy products as the layers land.

---

## Bottom line

The current platform is a strong **data plane** with a prototype intelligence layer. The 12-month target keeps Databricks as that data/ML plane, lifts reasoning and access into **governed services over a bitemporal contract knowledge graph**, and adds the five layers an enterprise legal-AI system actually requires but this one lacks today: **governance, observability, evaluation, trust, and legal-reasoning/semantic-state.** The order matters — measurement and state first, then retrieval and trust, then spend-weighted intelligence and production hardening — because every later capability depends on a single, correct, auditable notion of what is contractually in force.

---

*Synthesis of Phases 1–7; no new repository scan. Evidence for each current-state claim is in the corresponding phase document in this folder.*
