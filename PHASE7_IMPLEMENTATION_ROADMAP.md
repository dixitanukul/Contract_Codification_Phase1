# PHASE 7 — Enterprise Implementation Roadmap
## Contract Intelligence Platform — From Extraction Pipeline to Trusted Enterprise System
**Date:** May 19, 2026  
**Role:** Principal Enterprise AI Program Architect / Technical Product Strategist / Enterprise Legal AI Delivery Lead  
**Basis:** Phases 1-6 findings, live platform metrics (3,519 extractions, 69K rates, 278 canonical providers, 33.9% corpus coverage)

---

## 1. Executive Summary

The Contract Codification Pipeline has proven the feasibility of large-scale provider contract extraction. It now stands at an inflection point: **33.9% of the corpus is processed**, the data model is stabilizing, and the extraction quality is measurable (if imperfect). The question is no longer "can this work?" — it is "what sequence of investments transforms this into a trusted enterprise system?"

This roadmap answers that question with a delivery-optimized sequence that prioritizes:

1. **Compliance blockers first** — PII masking and audit trail (Week 1-2)
2. **Trust instrumentation next** — provenance, versioning, monitoring (Week 2-4)
3. **Quality measurement then** — golden dataset + precision/recall benchmarking (Month 2-3)
4. **Scale processing** — complete the remaining 6,853 files with improved quality (Month 2-4)
5. **Retrieval capabilities last** — semantic search only after structured data is trusted (Month 4-6)

**The strategic principle:** Do not build retrieval on top of untrusted data. Fix the data trust foundation first, then layer semantic capabilities on top.

### Current Platform Metrics

| Metric | Value |
|--------|-------|
| Files extracted | 3,519 / 10,372 (33.9%) |
| Consolidated providers | 456 |
| Canonical providers resolved | 278 |
| Total rate rows | 69,416 |
| Current rates (serving) | 12,283 |
| Clause/term/definition rows | 69,529 |
| Party records | 11,478 |
| Delta table versions | 185 |
| PII records unmasked | 2,043 (1,211 Tax IDs + 832 NPIs) |
| Production monitoring | NONE |
| Golden dataset size | 0 documents |
| Regression test coverage | 0% |

---

## 2. Strategic Architecture Direction

### 2.1 The Three-Phase Evolution

```
CURRENT STATE          TARGET STATE (6 months)     VISION STATE (12 months)
================       =======================     ========================
Extraction Pipeline    Governed Data Product        Contract Intelligence Platform
                       
* Notebook-driven      * Job-orchestrated          * Hybrid SQL + Semantic
* Dev-only             * Dev/Test/Prod catalogs    * Grounded legal QA
* No monitoring        * Full observability        * Citation-linked answers
* No review gates      * Human review for risk     * Self-improving evaluation
* PII exposed          * Masked + governed         * Full compliance posture
* 33.9% processed      * 100% processed            * Continuous ingestion
* No golden data       * 75-doc benchmark          * Growing benchmark
* Trust assumed        * Trust computed            * Trust enforced
```

### 2.2 Architecture Principles (Ordered by Priority)

1. **Compliance before features** — mask PII before adding any new capabilities
2. **Measure before improve** — instrument trust metrics before optimizing extraction
3. **Deterministic before probabilistic** — strengthen rule-based validation before adding AI verification
4. **Breadth before depth** — process all 10,372 files before perfecting extraction on a subset
5. **Structured before semantic** — SQL analytics must be trusted before RAG is built on top
6. **Parallel evaluation plane** — evaluation never blocks extraction, but gates deployment

---

## 3. Highest-Risk Areas

### 3.1 Risk Prioritization Matrix

| # | Risk | Probability | Impact | Urgency | Current Mitigation |
|---|------|------------|--------|---------|-------------------|
| 1 | **HIPAA violation from PII exposure** | HIGH (data exists today) | CRITICAL (regulatory fine) | IMMEDIATE | NONE |
| 2 | **Silent model regression** | MEDIUM (model updates unpredictable) | HIGH (all future extractions degraded) | HIGH | NONE |
| 3 | **Rate-service misattribution** | HIGH (est. 15%) | HIGH (financial decisions) | HIGH | NONE |
| 4 | **Workspace file loss** (json_extract/) | LOW-MEDIUM | CATASTROPHIC (no backup) | HIGH | NONE |
| 5 | **Data freshness decay** | MEDIUM (no schedule) | MEDIUM (stale rates) | MEDIUM | NONE |
| 6 | **Overwrite with corrupt data** | LOW-MEDIUM | HIGH (no rollback beyond 30 days) | MEDIUM | Delta time-travel (30d default) |
| 7 | **Incomplete corpus** (6,853 files remaining) | CERTAIN | MEDIUM (partial intelligence) | MEDIUM | In-progress |
| 8 | **No audit trail for governance** | HIGH (no run logging) | MEDIUM (compliance gap) | HIGH | NONE |

### 3.2 Risk Mitigation Sequence

The roadmap addresses risks in impact-weighted urgency order:
1. PII masking (Risk #1) — Week 1
2. Audit trail + versioning (Risk #8) — Week 1-2
3. Data durability (Risk #4) — Week 2
4. Retention extension (Risk #6) — Week 1
5. Regression detection (Risk #2) — Week 3-4
6. Production scheduling (Risk #5) — Week 3
7. Attribution evaluation (Risk #3) — Month 2
8. Full corpus processing (Risk #7) — Month 2-4

---

## 4. Highest-ROI Improvements

### 4.1 ROI Analysis

| Initiative | Effort | Trust Impact | Business Value | ROI Score |
|-----------|--------|-------------|----------------|-----------|
| PII column masking | 2 days | +0.3 (compliance) | Regulatory risk elimination | **CRITICAL** |
| Model/prompt versioning in JSON | 1 day | +0.2 (reproducibility) | Enables regression detection | **9.5/10** |
| Production monitoring job | 2 days | +0.2 (observability) | Early anomaly detection | **9.0/10** |
| Delta retention extension | 1 hour | +0.1 (data safety) | 365-day rollback capability | **9.0/10** |
| Golden dataset (20 docs) | 2 weeks | +0.3 (measurement) | First precision/recall numbers | **8.5/10** |
| Scheduled pipeline execution | 1 day | +0.1 (freshness) | Eliminates manual triggering | **8.5/10** |
| Rate attribution evaluation | 3 days | +0.2 (trust) | Quantifies largest error class | **8.0/10** |
| Hallucination source check | 3 days | +0.2 (trust) | Catches fabricated values | **8.0/10** |
| Full corpus extraction | 4-5 weeks | +0.1 (completeness) | 3x data coverage | **7.5/10** |
| Genie Space configuration | 2 days | +0.1 (accessibility) | Immediate analyst self-service | **7.5/10** |
| Human review queue | 2 weeks | +0.2 (trust) | Risk-gated publishing | **7.0/10** |
| Vector search on clauses | 1 week | +0.1 (capability) | Semantic legal search | **6.0/10** |
| Hybrid retrieval router | 2 weeks | +0.1 (capability) | Unified query experience | **5.5/10** |

### 4.2 The "80/20" Insight

**80% of enterprise trust improvement comes from 5 actions taking <2 weeks total:**

1. Mask PII columns (2 days)
2. Add model_version + extraction_timestamp to JSON metadata (1 day)
3. Create pipeline run log table (1 day)
4. Extend Delta retention to 365 days (1 hour)
5. Create production monitoring job (2 days)

These are not glamorous. They are not AI. They are governance fundamentals that transform the platform from "interesting prototype" to "auditable enterprise system."

---

## 5. Recommended Implementation Sequence

### 5.1 Dependency Graph

```
Week 1-2: COMPLIANCE & INSTRUMENTATION
  [PII Masking] ─────────────────────────────────────────────── (standalone)
  [Retention Extension] ─────────────────────────────────────── (standalone)
  [Model/Prompt Versioning] ──┐
  [Run Log Table] ────────────┼──> [Regression Detection] (Week 3-4)
  [Extraction Timestamp] ─────┘         │
                                        v
Week 3-4: MONITORING & SCHEDULING                               
  [Production Monitoring Job] ──> [Drift Detection] (Month 2)
  [Scheduled Pipeline Execution] ──> [Full Corpus Processing] (Month 2-4)
  [Genie Space Config] ─────────────────────────────────────── (standalone)
                                        
Month 2-3: MEASUREMENT & QUALITY                               
  [Golden Dataset - 20 docs] ──> [Precision/Recall Pipeline]
  [Rate Attribution Eval] ──────> [Attribution Improvement]
  [Hallucination Source Check] ─> [Confidence Scoring]
  [Full Corpus Batch 1] ─────────────────────────────────────
                                        
Month 3-4: TRUST & REVIEW                                      
  [Golden Dataset - 50 docs] ──> [Benchmark Gates]
  [Human Review Queue] ──────────> [Review Calibration]
  [Trust Scoring Pipeline] ──────> [Trust Dashboard]
  [Full Corpus Batch 2-3] ────────────────────────────────────
                                        
Month 4-6: RETRIEVAL & INTELLIGENCE                            
  [Clause Embeddings] ──> [Vector Search Index]
  [Query Router] ────────> [Hybrid Retrieval]
  [Citation Architecture] ──> [Grounded QA]
  [Amendment Graph] ──────> [Historical State Materialization]
```

### 5.2 Critical Path

The critical path is: **PII Masking → Run Logging → Model Versioning → Golden Dataset → Precision/Recall → Regression Gates → Production Qualification**

Everything else (retrieval, RAG, Genie) can proceed in parallel but does NOT gate production readiness.

---

## 6. Immediate Next Actions (This Week)

### Action 1: Mask PII Columns
**What:** Apply column masks to `tax_id` and `npi` in `dev_adb.raw.tbl_contract_parties`
**Why:** 2,043 PII records in cleartext = immediate compliance exposure
**How:**
```sql
-- Create masking function
CREATE OR REPLACE FUNCTION dev_adb.raw.mask_pii(val STRING)
  RETURNS STRING
  RETURN CASE WHEN is_account_group_member('pii_readers') THEN val
              ELSE CONCAT('***-', RIGHT(val, 4)) END;

-- Apply masks
ALTER TABLE dev_adb.raw.tbl_contract_parties
  ALTER COLUMN tax_id SET MASK dev_adb.raw.mask_pii;
ALTER TABLE dev_adb.raw.tbl_contract_parties
  ALTER COLUMN npi SET MASK dev_adb.raw.mask_pii;
```
**Owner:** Platform engineer
**Duration:** 2 hours (+ group creation)
**Success metric:** `SELECT tax_id FROM tbl_contract_parties LIMIT 1` returns masked value for non-privileged users

### Action 2: Extend Delta Retention
**What:** Set `delta.deletedFileRetentionDuration` to 365 days on all 35 tables
**Why:** Default 30-day retention means data loss is irrecoverable after 1 month
**How:**
```sql
ALTER TABLE dev_adb.raw.tbl_contract_rates_all 
  SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = '365 days');
-- Repeat for all tables
```
**Owner:** Platform engineer
**Duration:** 30 minutes
**Success metric:** All tables show 365-day retention in TBLPROPERTIES

### Action 3: Add Model/Prompt Metadata to Extraction
**What:** Record `model_name`, `model_version`, `prompt_version`, `extraction_timestamp` in every JSON file_metadata
**Why:** Cannot detect regression without knowing which model produced which extraction
**How:** Modify notebook 05_llm_extraction to append metadata fields after each API call
**Owner:** Pipeline developer
**Duration:** 2 hours
**Success metric:** New extractions have model metadata; `file_metadata.model_name` populated

### Action 4: Create Pipeline Run Log Table
**What:** Create `dev_adb.raw.ctl_pipeline_runs` recording every execution
**Schema:**
```sql
CREATE TABLE dev_adb.raw.ctl_pipeline_runs (
  run_id STRING,
  run_timestamp TIMESTAMP,
  triggered_by STRING,
  notebook_path STRING,
  mode STRING,  -- 'sample' or 'full'
  files_processed INT,
  files_failed INT,
  duration_seconds DOUBLE,
  model_endpoint STRING,
  prompt_version STRING,
  parameters MAP<STRING, STRING>,
  status STRING,  -- 'success' | 'partial' | 'failed'
  error_summary STRING
);
```
**Owner:** Pipeline developer
**Duration:** 3 hours
**Success metric:** Next pipeline run creates a row in ctl_pipeline_runs

### Action 5: Backup JSON Extractions to Volume
**What:** Copy json_extract/ and json_consolidated/ to a Unity Catalog Volume
**Why:** Workspace files have no replication; loss = re-extract 3,519 files (~5 days LLM time)
**How:**
```python
dbutils.fs.cp(
  "file:/Workspace/Users/adixit01@blueshieldca.com/Contract_Codification_Pipeline/json_extract/",
  "/Volumes/dev_adb/raw/contract_artifacts/json_extract/",
  recurse=True
)
```
**Owner:** Platform engineer
**Duration:** 1 hour
**Success metric:** Volume contains all JSON files; checksums match

---

## 7. 30-Day Plan

### Week 1: Compliance & Instrumentation

| Day | Task | Owner | Deliverable |
|-----|------|-------|-------------|
| 1 | Apply PII column masks | Platform eng | Masked tax_id, npi |
| 1 | Extend Delta retention to 365d (all tables) | Platform eng | Updated TBLPROPERTIES |
| 2 | Add model/prompt version to extraction code | Pipeline dev | Modified notebook 05 |
| 2 | Create ctl_pipeline_runs table | Pipeline dev | Table created + write logic |
| 3 | Backup JSON files to UC Volume | Platform eng | Volume with 3,519 files |
| 3 | Create ctl_prompt_versions table | Pipeline dev | Prompt text + version stored |
| 4-5 | Add extraction_timestamp to file_metadata | Pipeline dev | Modified extraction code |
| 4-5 | Add provenance_type column to key tables | Pipeline dev | program_provenance, date_provenance |

### Week 2: Monitoring Foundation

| Day | Task | Owner | Deliverable |
|-----|------|-------|-------------|
| 6-7 | Create production monitoring notebook | Platform eng | Data quality checks |
| 8 | Create eval schema (dev_adb.eval) | Platform eng | Schema + initial tables |
| 9 | Schedule monitoring as daily job | Platform eng | Lakeflow Job running |
| 10 | Configure Genie Space on genie tables | Analyst | Genie space active |

**Monitoring notebook checks:**
- Row count stability (±10% alert)
- NULL rate spikes (>5% alert)
- rate_numeric distribution drift (KS test)
- Provider count stability
- Pipeline last-run freshness
- Unprocessed files in Volume

### Week 3: Scheduling & Regression Foundation

| Day | Task | Owner | Deliverable |
|-----|------|-------|-------------|
| 11-12 | Convert pipeline to scheduled Lakeflow Job | Platform eng | Job created, weekly schedule |
| 13 | Implement model version detection on startup | Pipeline dev | Alert if model changed |
| 14-15 | Design golden dataset annotation template | QA lead | Annotation guide + template |

### Week 4: Golden Dataset Kickoff + Regression Baseline

| Day | Task | Owner | Deliverable |
|-----|------|-------|-------------|
| 16-17 | Select 20 golden dataset candidates (stratified) | QA lead | 20 files selected |
| 18-20 | Begin annotation of first 10 documents (rates) | Contract analyst | 10 docs with rate labels |
| 18-20 | Store current extraction baseline for golden docs | Pipeline dev | Baseline snapshot |

### 30-Day Success Criteria

| Criterion | Target |
|-----------|--------|
| PII masked | YES |
| Retention extended | YES (365 days) |
| Run log table populated | ≥1 row |
| Model version recorded | YES in new extractions |
| JSON backed up to Volume | 3,519 files |
| Monitoring job running | Daily |
| Genie Space configured | YES |
| Pipeline scheduled | Weekly |
| Golden documents selected | 20 |
| Golden annotations started | 10 docs in progress |
| Provenance columns added | program_provenance, date_provenance |

---

## 8. 90-Day Plan

### Month 2: Measurement & Scale

**Track A: Evaluation Foundation**

| Week | Task | Deliverable |
|------|------|-------------|
| 5-6 | Complete golden dataset (20 docs — rates, dates, clauses) | eval_golden_rates, eval_golden_clauses populated |
| 5-6 | Implement extraction precision/recall pipeline | MLflow experiment with first P/R/F1 metrics |
| 7 | Implement rate-service attribution evaluation | Attribution accuracy score computed |
| 7 | Implement automated hallucination source check | Source verification rate computed |
| 8 | Implement clause completeness scoring | Clause completeness metric active |

**Track B: Scale Processing**

| Week | Task | Deliverable |
|------|------|-------------|
| 5-8 | Process next 2,000 files (batches of 500) | json_extract count: ~5,500 |
| 5-8 | Monitor extraction quality per batch | Per-batch coverage scores tracked |
| 7-8 | Re-build all Delta tables from expanded corpus | Tables reflect 5,500 files |

**Track C: Operational Hardening**

| Week | Task | Deliverable |
|------|------|-------------|
| 5 | Implement drift detection in monitoring | Distribution checks active |
| 6 | Add alerting (webhook/email on SLA breach) | Alerts configured |
| 7 | Implement freshness SLA check | Stale data alert if >7 days |
| 8 | Create evaluation dashboard | Dashboard showing trust metrics |

### Month 3: Quality Gates & Trust

**Track A: Evaluation Maturity**

| Week | Task | Deliverable |
|------|------|-------------|
| 9-10 | Expand golden dataset to 50 documents | Full benchmark suite |
| 9 | Implement regression test suite | Regression gates defined |
| 10 | Run first model regression test | Baseline metrics stored |
| 11 | Implement document-level trust scoring | Trust scores on all docs |
| 12 | Implement platform trust score | Executive trust metric |

**Track B: Continue Scale Processing**

| Week | Task | Deliverable |
|------|------|-------------|
| 9-12 | Process remaining ~4,800 files | Target: 100% extraction |
| 10 | Re-build serving tables at full scale | 10K+ files in tables |
| 11-12 | Run full-corpus quality assessment | Corpus-wide metrics |

**Track C: Human Review Foundation**

| Week | Task | Deliverable |
|------|------|-------------|
| 9-10 | Design review queue schema + routing rules | Review architecture |
| 11-12 | Build review queue MVP (high-risk items only) | Review app/notebook |

### 90-Day Success Criteria

| Criterion | Target |
|-----------|--------|
| Golden dataset | 50 documents annotated |
| Extraction precision (rates) | Measured (target >85%) |
| Extraction recall (rates) | Measured (target >80%) |
| Rate attribution accuracy | Measured (target >85%) |
| Hallucination source check | Automated, running per extraction |
| Files processed | >90% of corpus (>9,300) |
| Regression baseline | Stored for golden dataset |
| Platform trust score | Computed and tracked |
| Monitoring + alerting | Active with defined SLAs |
| Human review queue | MVP operational |
| Evaluation dashboard | Live with key metrics |
| Trust score | Computed (target >0.65) |

---

## 9. 6-Month Architecture Plan

### Month 4-5: Retrieval & Intelligence Layer

**Prerequisite:** Structured data is trusted (precision >85%, monitoring active, review queue running)

| Initiative | Description | Databricks Service |
|-----------|-------------|-------------------|
| Clause embeddings | Embed 69K+ clause/definition/section rows | Model Serving |
| Vector Search index | Create index on clause embeddings with metadata filters | Vector Search |
| Query router MVP | Classify queries as structured/semantic/hybrid | Python + rules |
| Hybrid retrieval service | SQL + vector results merged with citations | SQL + Vector Search |
| Citation architecture | Link answers to source_filename + page_number | Delta metadata |

**Key decisions for Month 4:**
- Embedding model selection (BGE-large vs E5 vs domain-specific)
- Index strategy: single index with filters vs separate indices per content_type
- Chunk granularity: clause-level vs section-level vs both
- Metadata filters: provider_id, content_type, is_from_latest_doc, clause_type

### Month 5-6: Canonical Contract Model

| Initiative | Description |
|-----------|-------------|
| Contract family entity | Group all documents for one provider agreement |
| Amendment graph | Model supersession relationships between documents |
| Historical state materialization | Reconstruct contract state at any point in time |
| Deterministic supersession engine | Rule-based current-state computation (not LLM) |
| Improved chunking | Structure-aware segmentation (table/clause/section) |

### Month 6: Enterprise Readiness Gate

| Gate Requirement | Pass Criteria |
|-----------------|---------------|
| Extraction precision | >90% on 75-doc golden set |
| Extraction recall | >85% on 75-doc golden set |
| Hallucination rate | <5% on golden set |
| Rate attribution accuracy | >87% |
| Platform trust score | >0.75 |
| PII protection | All columns masked |
| Production monitoring | >30 days of clean operation |
| Human review | Queue active, SLAs met |
| Regression tests | Passing for current model/prompt |
| Data retention | 365-day policy active |
| Audit trail | All runs logged |
| Freshness SLA | <7 days data currency |
| Full corpus | 100% processed |
| Documentation | Architecture, evaluation, operations docs complete |

---

## 10. 12-Month Vision

### Month 7-9: Legal Intelligence Platform

| Capability | Description |
|-----------|-------------|
| Grounded legal QA | Answer legal questions with citations and confidence |
| Clause comparison | Semantic comparison across providers |
| Amendment impact analysis | "What changed and why" for each amendment |
| Contract portfolio dashboard | Executive view of rate landscape |
| Rate anomaly detection | Statistical outliers + temporal anomalies |

### Month 10-12: Self-Improving System

| Capability | Description |
|-----------|-------------|
| Continuous evaluation | Automated quality tracking per batch |
| Calibrated confidence | Trust scores validated by human review |
| Multi-model support | Compare extraction across model vendors |
| Auto-ingestion | New PDFs trigger extraction automatically |
| Legal knowledge graph | Clause topics, obligations, cross-references |
| Contract negotiation intelligence | Historical rate trends + clause patterns |

### What the Platform Looks Like at Month 12

- **Every contract** for all 597 providers is extracted, validated, and served
- **Every rate** has a trust tier (HIGH/MEDIUM/LOW) based on source verification + human review
- **Every clause** is semantically searchable with citations
- **Every amendment** is linked in a supersession graph
- **Every answer** to a legal question cites its source with page numbers
- **Analysts** self-serve via Genie for structured questions
- **Legal operations** uses semantic search for clause analysis
- **Executives** see a trust-scored contract portfolio dashboard
- **Compliance** has full audit trail, masking, and retention

---

## 11. Recommended Team Structure

### Minimal Viable Team (Months 1-3)

| Role | FTE | Responsibilities |
|------|-----|-----------------|
| **Pipeline Developer** | 1.0 | Extraction code, prompt engineering, scale processing |
| **Platform/Data Engineer** | 0.5 | Governance, monitoring, scheduling, infrastructure |
| **Contract Analyst** (domain expert) | 0.5 | Golden dataset annotation, review queue |

### Growth Team (Months 4-6)

| Role | Additional FTE | Responsibilities |
|------|---------------|-----------------|
| **ML/AI Engineer** | 0.5 | Embeddings, vector search, retrieval quality |
| **QA/Evaluation Specialist** | 0.5 | Evaluation pipelines, regression testing |

### Full Team (Months 7-12)

| Role | Additional FTE | Responsibilities |
|------|---------------|-----------------|
| **Product Owner** | 0.5 | Prioritization, stakeholder management |
| **Legal SME** | 0.25 | Adjudication, legal accuracy validation |

### Ownership Model

| Area | Owner | Accountable For |
|------|-------|----------------|
| Extraction quality | Pipeline Developer | Precision, recall, coverage |
| Platform reliability | Platform Engineer | Uptime, monitoring, governance |
| Data trust | QA Specialist | Trust scores, regression gates |
| Business value | Product Owner | Stakeholder satisfaction, adoption |
| Domain correctness | Contract Analyst | Golden dataset, review decisions |

---

## 12. Recommended Governance Process

### 12.1 Change Management

| Change Type | Review Required | Gate |
|------------|----------------|------|
| Prompt modification | Regression test on golden set | P/R must not decrease >2% |
| Model endpoint change | Full regression suite | All gates pass |
| Schema change | Code review + backward compatibility check | No breaking changes |
| New table creation | Architecture review | Naming + governance compliance |
| Golden dataset modification | Dual-annotator review | IAA > 0.80 |
| Trust threshold change | Stakeholder sign-off | Documented justification |

### 12.2 Release Process

```
Development → Regression Test → Staging Extraction → Quality Gate → Production
                    │                                       │
                    │ FAIL: Block + investigate              │ FAIL: Block + review
                    v                                       v
              Fix + retest                          Stakeholder decision
```

### 12.3 Operational Cadence

| Activity | Frequency | Participants |
|----------|-----------|-------------|
| Pipeline run | Weekly (scheduled) | Automated |
| Monitoring review | Daily (automated checks) | Platform eng (alerts only) |
| Quality review | Bi-weekly | Pipeline dev + QA |
| Golden dataset expansion | Monthly | Contract analyst + QA |
| Trust score reporting | Monthly | Platform eng + Product |
| Architecture review | Quarterly | Full team |
| Compliance audit | Quarterly | Platform eng + Legal |

---

## 13. Recommended Productionization Strategy

### 13.1 Environment Strategy

| Environment | Catalog | Purpose | Access |
|------------|---------|---------|--------|
| **Development** | `dev_adb` | Current state — active development | Pipeline developer |
| **Test/Staging** | `test_adb` (create) | Regression testing, golden dataset eval | QA + Platform eng |
| **Production** | `prod_adb` (create) | Trusted serving tables for consumption | Analysts + Apps |

### 13.2 Promotion Path

1. Extraction runs in `dev_adb` with full monitoring
2. Quality gates evaluate against golden dataset
3. If gates pass: rebuild serving tables in `prod_adb`
4. Production tables are READ-ONLY for consumers
5. No direct writes to prod; only governed promotion

### 13.3 Suggested Lakeflow Jobs

| Job | Schedule | Tasks |
|-----|----------|-------|
| `contract_extraction_weekly` | Sunday 2AM | Discovery → OCR → Extract → Validate → Build tables |
| `contract_monitoring_daily` | Daily 6AM | Quality checks → drift detection → freshness → alerts |
| `contract_evaluation_weekly` | Monday 8AM | Golden benchmark → trust scores → regression check |
| `contract_review_routing` | Daily 9AM | Route high-risk items to review queue |

---

## 14. Recommended Technical Milestones

### Milestone 1: "Auditable" (Day 30)
- Every extraction records model version and timestamp
- Every run is logged
- PII is masked
- Data has 365-day retention
- Platform has daily monitoring

### Milestone 2: "Measurable" (Day 90)
- Extraction precision/recall quantified on golden set
- Hallucination rate measured
- Trust score computed
- Regression baseline stored
- Full corpus processed

### Milestone 3: "Trusted" (Day 180)
- Precision >90%, recall >85%
- Human review active for high-risk items
- Regression gates block degradation
- Platform trust score >0.75
- Genie Space active for analyst consumption

### Milestone 4: "Intelligent" (Day 365)
- Semantic search over clauses
- Grounded legal QA with citations
- Amendment lineage graph
- Historical state reconstruction
- Self-improving evaluation loop

---

## 15. Critical Architectural Warnings

### Warning 1: Do Not Build Retrieval on Untrusted Data
If the structured extraction has 15% rate misattribution, a RAG system built on top will confidently return wrong answers with citations pointing to incorrect data. **Fix data quality first, then build retrieval.**

### Warning 2: Do Not Overwrite Without Backup
The current `mode("overwrite").option("overwriteSchema", "true")` pattern is acceptable during development but dangerous in production. Before any production promotion:
- Always verify new data before overwriting
- Maintain at minimum Delta time-travel capability
- Consider append-then-swap for critical tables

### Warning 3: Do Not Skip the Golden Dataset
Every other evaluation mechanism (automated hallucination detection, drift monitoring, trust scoring) is calibrated AGAINST the golden dataset. Without it, you cannot distinguish a well-calibrated system from one that is confidently wrong.

### Warning 4: Do Not Attempt Multi-Model Serving Before Single-Model is Stable
Adding a second extraction model (for comparison or fallback) sounds valuable but doubles operational complexity. Master one model's behavior first, including prompt regression and output schema stability.

### Warning 5: Do Not Build GraphRAG
For this corpus, provider-contract-amendment relationships are fully expressible as SQL JOINs on the canonical contract model. GraphRAG adds massive complexity with minimal incremental value over SQL + vector hybrid retrieval. The amendment lineage graph should be a Delta table with foreign keys, not a graph database.

### Warning 6: Do Not Fine-Tune Embeddings
The clause corpus (69K units) is large enough for effective retrieval with general-purpose embedding models. Fine-tuning requires labeled relevance pairs, introduces training drift risk, and delays deployment. Use off-the-shelf embeddings (BGE-large or E5) with metadata filtering — this will outperform fine-tuned embeddings on a small corpus.

### Warning 7: Do Not Build a Custom Review UI Before Testing with Notebooks
A full review application (Databricks App) is the right long-term answer, but building it before validating the review workflow is premature. Start with a notebook-based review workflow to validate routing rules and reviewer workflows, then invest in UI.

---

## 16. What NOT To Build Yet

| Capability | Why Not Now | When to Revisit |
|-----------|------------|-----------------|
| **GraphRAG** | SQL JOINs handle all entity relationships; graph adds complexity | Never (for this use case) |
| **Custom embedding fine-tuning** | Corpus too small; general embeddings + metadata filtering sufficient | Month 9+ if retrieval quality < target |
| **Multi-modal retrieval** | OCR already handles all visual content | Not needed |
| **Agentic workflows** | Adds non-determinism; premature for legal trust | Month 9+ after QA framework is mature |
| **Real-time ingestion** | New contracts arrive monthly, not real-time | Month 9+ |
| **Cross-organization benchmarking** | No external data; internal optimization first | Year 2 |
| **Custom LLM fine-tuning** | Prompt engineering + general model sufficient; fine-tuning risks overfitting | Month 12+ if prompt ceiling reached |
| **Re-ranking models** | Corpus size (69K) doesn't warrant heavy re-ranking | Month 9+ if retrieval P@5 < 0.7 |
| **Full CI/CD pipeline** | Premature; operational discipline needed first | Month 4+ after regression suite stable |
| **PDF page image rendering for UI** | Nice-to-have; citation by filename + page sufficient initially | Month 6+ |

---

## 17. Final Strategic Recommendation

### The One Sentence

**Build trust infrastructure before building AI capabilities — the platform's value is not in generating answers, but in generating answers that enterprises can act on.**

### The Strategy in Three Moves

**Move 1 (Months 1-3): Make it auditable and measurable.**
Transform from an opaque extraction pipeline into a measured, monitored, governed system. This is not exciting, but it is the foundation everything else depends on. Key outputs: PII masked, runs logged, golden dataset created, precision/recall known, monitoring active, full corpus processed.

**Move 2 (Months 3-6): Make it trusted and reviewable.**
Layer trust scoring, human review, regression gates, and quality dashboards on top of the measurement foundation. Key outputs: platform trust score >0.75, review queue active, regression gates blocking, enterprise readiness gate passed.

**Move 3 (Months 6-12): Make it intelligent and accessible.**
Build semantic retrieval, grounded QA, and legal intelligence capabilities on top of the trusted data foundation. Key outputs: clause-level semantic search, citation-grounded answers, amendment lineage, portfolio analytics, self-improving evaluation.

### Why This Sequence

The temptation is to jump to Move 3 (it's the most impressive). But an AI system that gives semantically impressive answers based on 15% misattributed data is worse than no system at all — it creates confident wrong decisions. The correct sequence is:

1. **Know what you have** (instrumentation)
2. **Know how good it is** (measurement)
3. **Make it better** (improvement)
4. **Make it accessible** (retrieval)
5. **Make it intelligent** (reasoning)

Each layer depends on the one below it. Skip a layer and the system will appear to work but will eventually fail in ways that destroy enterprise trust permanently.

### The North Star Metric

**Platform Trust Score > 0.85** — computed as a weighted composite of extraction precision, source verification rate, human review agreement, production SLA compliance, and regression gate status. When this metric is achieved and sustained for 30 consecutive days, the platform is ready for enterprise legal reliance.

---

## Appendix: Initiative Detail Cards

### Initiative: Golden Dataset Creation
- **Business value:** HIGH — enables all quality measurement
- **Technical value:** CRITICAL — gates precision/recall/regression
- **Trust impact:** +0.30 — transforms trust from qualitative to quantitative
- **Complexity:** MEDIUM (requires domain expert annotation time)
- **Dependencies:** Domain expert availability, document selection
- **Risks:** Annotator inconsistency; selection bias
- **Databricks services:** Delta tables in dev_adb.eval
- **Data model changes:** New eval schema with golden tables
- **Evaluation support:** IS the evaluation foundation
- **Rollout:** Start with 10 rate-heavy docs, expand monthly
- **Ownership:** Contract analyst (annotate) + QA specialist (process)
- **Success metrics:** 50 docs annotated with IAA > 0.80

### Initiative: Rate Attribution Evaluation
- **Business value:** HIGH — quantifies the #1 financial risk
- **Technical value:** HIGH — validates most important extraction
- **Trust impact:** +0.20 — makes invisible error visible
- **Complexity:** MEDIUM (requires golden rate labels)
- **Dependencies:** Golden dataset (rate layer)
- **Risks:** Initial attribution accuracy may be disturbingly low
- **Databricks services:** Notebooks + Delta eval tables
- **Data model changes:** eval_extraction_results table
- **Evaluation support:** Core extraction evaluation pipeline
- **Rollout:** Run on golden set first, then sample-based monitoring
- **Ownership:** Pipeline developer (eval code) + QA (analysis)
- **Success metrics:** Attribution accuracy measured, target >85%

### Initiative: Vector Search on Clauses
- **Business value:** MEDIUM — unlocks 30% semantic query gap
- **Technical value:** HIGH — new capability
- **Trust impact:** +0.10 initially (retrieval must be evaluated)
- **Complexity:** MEDIUM (Databricks Vector Search is managed)
- **Dependencies:** Trusted clause data, embedding model selection
- **Risks:** Poor retrieval quality if clause data is incomplete; over-reliance before evaluation
- **Databricks services:** Vector Search, Model Serving (embedding)
- **Data model changes:** Embedding columns or managed index
- **Evaluation support:** Retrieval P@k, recall@k on query benchmark
- **Rollout:** Deploy on genie_contract_terms (69K rows), evaluate before exposing
- **Ownership:** ML engineer
- **Success metrics:** Precision@5 > 0.80 on benchmark queries

### Initiative: Hybrid Retrieval Router
- **Business value:** HIGH — unified analyst experience
- **Technical value:** HIGH — architectural cornerstone
- **Trust impact:** +0.10 — only if retrieval is grounded
- **Complexity:** HIGH (classification + orchestration + citation)
- **Dependencies:** Vector search operational, SQL serving reliable, citation links
- **Risks:** Mis-routing degrades trust; over-engineering
- **Databricks services:** Model Serving (router), SQL, Vector Search
- **Data model changes:** Query logs, retrieval metrics tables
- **Evaluation support:** Route accuracy, per-lane quality metrics
- **Rollout:** Start with deterministic rules, add ML classification later
- **Ownership:** ML engineer + Platform engineer
- **Success metrics:** Route accuracy >90%, end-to-end answer quality >0.85

---

## END OF PHASE 7 IMPLEMENTATION ROADMAP
