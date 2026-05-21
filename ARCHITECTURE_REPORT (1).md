# PHASE 1 — Repository & Architecture Reconstruction
## Contract Codification Pipeline — Complete Current-State Analysis
**Date:** May 19, 2026  
**Analyst Role:** Principal AI Architect, Principal Data Engineer, Enterprise GenAI Reviewer, Legal AI System Auditor  
**Repository Path:** `/Workspace/Users/adixit01@blueshieldca.com/Contract_Codification_Pipeline`

---

## 1. Repository Structure Breakdown

```
/Contract_Codification_Pipeline/
├── _config                          (shared configuration notebook — %run target)
├── 01_file_discovery                (PDF Volume scan + SSD prefetch + checkpoint)
├── 02_ocr_extraction                (2-tier text extraction: pypdf + ai_parse_document)
├── 03_metadata_parsing              (filename regex → provider_id, contract_id, doc_type, version)
├── 04_llm_prompts                   (4 document-type prompts + routing logic)
├── 05_llm_extraction                (REST API engine: chunking, adaptive concurrency, retry)
├── 06_validation                    (production gate + medical codes + rate/date verification)
├── 07_consolidation                 (per-provider JSON merge + amendment chain ordering)
├── 08_build_rates                   (base layer: rates_all + regional_factors + financial_protections)
├── 09_build_terms                   (base layer: clauses + codes + parties + services)
├── 10_build_documents               (base layer: documents_master + dim_provider_canonical)
├── 11_enrich_tables                 (13 idempotent DQ steps across all base tables)
├── 12_build_genie                   (4 Genie-layer tables with 67 column comments)
├── 13_build_serving                 (12 normalized serving-layer tables)
├── 14_quality_audit                 (LLM-based extraction gap detection)
├── 15_gap_filler                    (targeted re-extraction for identified gaps)
├── README_tables                    (user-facing documentation notebook)
├── EVALUATION_PROMPT.md             (10-phase evaluation protocol)
├── checkpoints/                     (4 parquet files: file_registry, step1, step2, scores)
├── json_extract/                    (~1,000 files — one JSON per processed PDF)
├── json_consolidated/               (~456 files — one JSON per provider)
└── json_extract_backup/             (pre-gap-fill originals)
```

**No Python modules, no utility `.py` files, no external libraries, no Wheel packages.** The entire codebase is self-contained within 16 Databricks notebooks + 1 config notebook shared via `%run`.

---

## 2. Current Pipeline Walkthrough (15 Numbered Steps)

The pipeline executes as a **linear sequential chain** (each notebook depends on the previous):

| Phase | Notebook | Internal Step # | Core Action | LLM? |
|-------|----------|-----------------|-------------|------|
| **Ingest** | 01 | Step 1a | Volume scan + SSD prefetch | No |
| **OCR** | 02 | Step 1b | 2-tier text extraction | Paid (ai_parse_document) |
| **Metadata** | 03 | Step 2 | Filename regex parsing | No |
| **Prompts** | 04 | — | Prompt templates (loaded by 05) | — |
| **Extraction** | 05 | Step 3 | Claude REST API (3 workers) | Yes |
| **Validation** | 06 | Steps 4–5 | DQ enrichment + quality report | No |
| **Consolidation** | 07 | Step 6 | Per-provider merge + dedup | No |
| **Build Rates** | 08 | Step 11a | 3 rate tables from JSON | No |
| **Build Terms** | 09 | Step 11b | 4 terms tables from JSON | No |
| **Build Docs** | 10 | Step 11c | Documents master + identity resolution | No |
| **Enrich** | 11 | Step 11c-enrich | 13 DQ fixups (idempotent) | No |
| **Genie Layer** | 12 | Step 11d | 4 AI-optimized tables | No |
| **Serving Layer** | 13 | Step 11e | 12 normalized analytics tables | No |
| **Audit** | 14 | — | LLM-based gap detection | Yes |
| **Gap Fill** | 15 | — | Targeted re-extraction | Yes |

**Total LLM invocations:** 3 points (Steps 3, 14, 15). All use `databricks-claude-sonnet-4-5` via direct REST POST.

---

## 3. Data Flow Mapping

```
/Volumes/prod_adb/default/.../Provider_Contracts/
    (10,372 PDFs, 27 GB, 597 provider folders)
         │
         ▼
┌─ 01: file_registry_v2.parquet (file metadata cache) ──────────────┐
│  Prefetch to /local_disk0/tmp/provider_contracts (NVMe SSD)        │
└────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ 02: step1_parsed_v2.parquet (OCR text + extraction_method) ──────┐
│  Tier 1: pypdf (82%) │ Tier 2: ai_parse_document (18%)             │
└────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ 03: step2_with_metadata_v2.parquet (text + parsed metadata) ─────┐
│  provider_id, provider_name, contract_id, document_type, version   │
└────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ 05: json_extract/*.json (one per PDF, ~1000 files processed) ────┐
│  V6 schema: contract_overview, rates, clauses, parties, services   │
│  + file_metadata + validation section (codes, scores, flags)       │
└────────────────────────────────────────────────────────────────────┘
         │
         ├──── 06: Validation enrichment (in-place JSON update)
         │
         ▼
┌─ 07: json_consolidated/*.json (one per provider, ~456 files) ─────┐
│  document_timeline, merged rates, amendment chains, supersession    │
└────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ 08–13: 35 Delta Tables in dev_adb.raw ──────────────────────────┐
│  BASE (8+3): tbl_contract_* + dim_provider_canonical + dims        │
│  GENIE (4):  tbl_genie_*                                           │
│  SERVING (12): tbl_serving_* + dim_service_category + dim_network  │
│  AUDIT (2):  tbl_extraction_quality_audit + _gaps                  │
└────────────────────────────────────────────────────────────────────┘
```

**Key observation:** The pipeline reads JSON files directly to build Delta tables (notebooks 08–09). It does **NOT** build tables from other tables — it always re-reads the source JSONs. Tables are rebuilt via `OVERWRITE` on each run (not incremental MERGE for base/genie/serving).

---

## 4. OCR & Parsing Architecture

### 2-Tier Design (Notebook 02)

| Tier | Tool | Trigger | Parallelism | Success Rate |
|------|------|---------|-------------|-------------|
| 1 | `pypdf` (PdfReader) | Default for all files | 14 OS processes via ProcessPoolExecutor | ~82% |
| 2 | `ai_parse_document` (Spark SQL) | Tier 1 < 500 chars OR < 100 chars/page | Batch of 25, 20 Spark partitions | ~18% |

**Critical design decisions:**
* GIL bypass via `ProcessPoolExecutor` (not threads) — 5.1x speedup measured
* Per-page timeout (30s) prevents hung PDFs from blocking the pool
* Encrypted PDF handling (attempts empty-password decrypt)
* Tier 2 uses pypdf partial text as fallback if ai_parse yields less
* **pytesseract OCR was explicitly removed** — too slow (150+ hrs) and lower quality than ai_parse_document

**Checkpointing:** After Tier 1 completes, results are immediately checkpointed to `step1_parsed_v2.parquet`. Tier 2 failures don't lose Tier 1 work.

---

## 5. LLM Architecture

### Model & Invocation

**Model:** `databricks-claude-sonnet-4-5` (200K context window)

**Invocation pattern:** Direct REST POST to `/serving-endpoints/databricks-claude-sonnet-4-5/invocations`
* Temperature: 0.0 (deterministic)
* Max output tokens: 65,536
* Per-file HTTP timeout: 1,800 seconds (30 min)
* No system prompt — uses single "user" message (prompt + document concatenated)

### Adaptive Concurrency (`AdaptiveConcurrency` class)
* Starts at 3 workers
* Backs off on 429/rate-limit detection (regex-matched)
* Rate limit detection regex: `429|rate.?limit|throttl|too many requests|capacity|overloaded|quota`

### Chunking Strategy
* Threshold: files > 150K chars (later adjusted to 270K — covers all corpus files single-call)
* Chunk size: 80,000 chars with 2,000 char overlap
* Results merged post-extraction

### JSON Repair
`_repair_truncated_json()` closes open brackets when output hits max_tokens

### 4 Document-Type Prompts
* `EXTRACTION_PROMPT` — Base Agreements (full schema)
* `AMENDMENT_PROMPT` — Amendments (supersession tracking focus)
* `COVER_MEMO_PROMPT` — Cover Memos (exhibits_replaced, rate escalators)
* `SETTLEMENT_PROMPT` — Settlements (settlement_amount, release scope)

### Routing Logic
`_get_prompt_category(doc_type)` — keyword matching on document_type string:
* `settlement` in name → Settlement prompt
* ends with `cm` or contains `covermemo` → Cover Memo prompt
* `amend` or `lra` or `addendum` → Amendment prompt
* everything else → Base Agreement prompt

### V6 Schema Mandates (embedded in every prompt)
* `rate_numeric` on every rate row (dollar as plain number)
* `full_clause_text` for legal clauses (verbatim, never summarize)
* `amendments_impact` for supersession tracking
* Dates normalized to YYYY-MM-DD
* `program` and `network` fields on every rate

---

## 6. Validation & Audit Architecture

### Layer 1 — Rule-Based Validation (Notebook 06)

| Check | Method |
|-------|--------|
| Medical codes | Regex: CPT (5-digit), HCPCS (alpha+4), ICD-10 (letter+2+dot+digits), Revenue (4-digit) |
| Date verification | Check if extracted date strings exist in raw PDF text (multi-format) |
| Rate verification | OCR-aware fuzzy matching (comma/period, missing $, whitespace variants) |
| rate_numeric fill | Post-processor: parses "$5,748" → 5748.00 where LLM missed |
| Coverage scoring | Weighted per section, doc-type-specific |
| Content classification | Bucket + stub detection (routing sheets ≠ failed extractions) |

### Layer 2 — LLM-Based Audit (Notebook 14)
* Sends both raw PDF text AND extracted JSON to Claude
* Taxonomy of gap types: missing_rate, missing_clause, incorrect_value, hallucinated_content, truncated_text
* Severity levels: HIGH, MEDIUM, LOW
* Outputs to 2 Delta tables via MERGE (tbl_extraction_quality_audit, tbl_extraction_quality_gaps)
* Supports 4 audit modes: all, single, sample, providers

### Layer 3 — Targeted Remediation (Notebook 15)
* Reads audit gaps from Delta
* Builds surgical prompt: "Here's existing JSON + known gaps → extract ONLY missing fields"
* Deep-merges new extractions into existing JSON (preserves prior work)
* Creates backup before overwrite
* Before/after validation comparison

---

## 7. Storage & Serving Architecture

### Storage Layout

| Layer | Storage | Pattern | Lifecycle |
|-------|---------|---------|----------|
| Source PDFs | Unity Catalog Volume (prod_adb) | Read-only, 27 GB | Incremental adds |
| Checkpoints | Workspace files (parquet) | Append-only, incremental | Rebuilt on new files |
| Per-file JSON | Workspace files | One per PDF, overwritten on gap-fill | Long-lived |
| Per-provider JSON | Workspace files | One per provider, rebuilt each run | Rebuilt |
| Delta Tables | Unity Catalog (dev_adb.raw) | OVERWRITE each run | Rebuilt |
| Backups | Workspace files | Pre-gap-fill originals | Append-only |

### Delta Table Write Strategy
* Base/Genie/Serving tables: `mode("overwrite").option("overwriteSchema", "true")` — full rebuild every run
* Audit tables: Delta MERGE (upsert by filename) — preserves historical audits
* No liquid clustering, no Z-ORDER, no optimize applied
* No table constraints or check constraints
* Schema evolution via `overwriteSchema=true`

### 35 Delta Tables in `dev_adb.raw`

| Layer | Count | Tables |
|-------|-------|--------|
| Base | 8 | rates_all, documents_master, financial_protections, regional_factors, clauses_terms, codes_events, parties, services_quality |
| Dimension | 3 | dim_provider_canonical, dim_service_category, dim_network |
| Genie | 4 | provider_profile, rates_current, contract_terms, amendment_timeline |
| Serving | 10 | provider_summary, rate_card, capitation_card, case_rate_card, outpatient_card, stop_loss, amendment_history, parties, regional_factors, services_coverage |
| Audit | 2 | extraction_quality_audit, extraction_quality_gaps |

---

## 8. Databricks Services Usage

| Service | Usage | Location |
|---------|-------|----------|
| **Unity Catalog Volumes** | Source PDF storage (prod_adb) | Notebook 01 |
| **ai_parse_document** | Tier 2 OCR (built-in Spark SQL) | Notebook 02 |
| **Model Serving Endpoint** | Claude Sonnet 4.5 (REST) | Notebooks 05, 14, 15 |
| **Delta Lake** | All 35 output tables | Notebooks 08–15 |
| **Databricks Widgets** | Runtime parameters (run_mode, force_rebuild, etc.) | _config, 14, 15 |
| **dbutils.notebook.entry_point** | API token for REST auth | _config |
| **Spark SQL** | Table creation, MERGE, UPDATE | Notebooks 10–13 |

### NOT Used (Notable Absences)
* No MLflow (no experiment tracking, no model registry)
* No Vector Search / embeddings
* No Genie Space configuration (tables are Genie-ready but no Genie space is created)
* No Unity Catalog governance tags or column masks
* No Delta Live Tables / SDP pipelines
* No scheduled jobs configured in this repo
* No streaming tables
* No Feature Store

---

## 9. Existing Retrieval/Search Understanding

**There is NO retrieval/search implementation.** The pipeline is purely extractive (PDF → structured data). No:
* Embedding generation
* Vector indexes
* Semantic search capability
* RAG pipeline
* Full-text search index

The closest analog is the `tbl_genie_contract_terms` table (69K rows of verbatim clause text) which could support keyword search via `ILIKE`, and the 67 column comments designed for Genie natural language queries. But no actual retrieval infrastructure exists.

---

## 10. Current Trust/Governance Mechanisms

### Hallucination Prevention

| Mechanism | Implementation | Notebook |
|-----------|----------------|----------|
| Temperature 0.0 | Deterministic output | 05 |
| Date verification | Check extracted dates exist in raw text | 06 |
| Rate verification | OCR-aware fuzzy match against source | 06 |
| Pre-1990 date nulling | Hallucinatory dates eliminated | 11 |
| LLM audit loop | Claude re-reads PDF to find extraction gaps | 14 |
| Source filename on every row | Full provenance tracing | 08, 09 |
| amendment_order tracking | Prevents superseded data appearing as current | 07, 08 |

### Data Quality Controls
* V6 coverage scoring (section-level completeness)
* Content bucket + stub detection (identifies routing sheets vs real contracts)
* rate_numeric post-processor (fills gaps where LLM missed)
* is_valid_rate flag (eliminates negative/zero rates)
* Sentinel value handling (amendment_order ≥ 100 excluded from counts)

### Auditability
* Every rate row carries `source_filename` — trace to exact PDF
* `extraction_method` column (pypdf vs ai_parse_document)
* Per-file backup before gap-fill overwrite
* Pipeline quality report with exit codes (0/1/2)
* `validation` key embedded in every JSON (schema_version, scores, flags)

### What's MISSING from a Governance Perspective
* No data classification or sensitivity tagging
* No access controls beyond schema-level
* No row-level security
* No column masking on PII (tax_id, NPI are stored in clear)
* No audit log table for who ran the pipeline or when
* No data retention policy
* No change data capture history (OVERWRITE destroys prior versions)

---

## 11. Current-State Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                    PROVIDER CONTRACT CODIFICATION PIPELINE                       │
│                    V8 — May 2026 — 35 Delta Tables                              │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                    │
│  │ INGEST       │     │ TRANSFORM    │     │ SERVE        │                    │
│  │              │     │              │     │              │                    │
│  │ 10,372 PDFs  │────▶│ Claude 4.5   │────▶│ 35 Delta     │                    │
│  │ Volume       │     │ REST API     │     │ Tables       │                    │
│  │ 27 GB        │     │ 3 workers    │     │ dev_adb.raw  │                    │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘                    │
│         │                    │                    │                             │
│  ┌──────▼───────┐     ┌──────▼───────┐     ┌──────▼───────┐                    │
│  │ 2-Tier OCR   │     │ Validation   │     │ Genie AI     │                    │
│  │ pypdf 82%    │     │ Rule + LLM   │     │ Dashboards   │                    │
│  │ ai_parse 18% │     │ Gap Fill     │     │ Analytics    │                    │
│  └──────────────┘     └──────────────┘     └──────────────┘                    │
│                                                                                 │
├────────────────────────────────────────────────────────────────────────────────┤
│  ORCHESTRATION: Manual sequential notebook execution (no job scheduler)         │
│  STATE: Parquet checkpoints (4 files) + JSON files (~1,456 total)               │
│  IDENTITY: dim_provider_canonical (456 IDs → 278 facilities)                    │
│  DEDUP: V8 amendment restatement dedup (27.6% → 3.7% duplicates)                │
│  TRUST: Source tracing + date/rate verification + LLM audit loop                │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Major Architectural Observations

### Strengths Observed

1. **Clean separation of concerns** — Each notebook does exactly one thing. Prompts are isolated from the extraction engine. Table build logic is separated from enrichment.
2. **Checkpoint-resumable** — The pipeline can crash mid-OCR or mid-extraction and resume from exactly where it left off (parquet + JSON checkpoints).
3. **LLM cost discipline** — REST API (not ai_query()) prevents cascade timeouts. Production gate skips rebuilds when no new files exist.
4. **V8 dedup innovation** — The amendment restatement problem (healthcare-specific) is solved elegantly by partitioning on rate_numeric rather than rate_text.
5. **Self-validating** — LLM audit + gap fill creates a closed loop: Extract → Audit → Re-extract gaps → Rebuild.
6. **Three-layer table model** — Clear progression from raw (full audit trail) → Genie (AI-optimized) → Serving (dashboard-ready).

### Architectural Risks and Gaps

1. **No orchestration** — Pipeline is run manually notebook-by-notebook. No Lakeflow Job, no dependency graph, no alerting on failure.
2. **OVERWRITE destroys history** — Base/Genie/Serving tables are fully rebuilt each run. If a bug is introduced, all prior correct data is lost. No time-travel safety net (Delta retains versions but no explicit governance).
3. **Workspace file storage for state** — `json_extract/` and `json_consolidated/` (~1,456 JSON files) are stored in workspace files, not a Volume or Delta table. This is fragile (no replication, no access control, workspace storage limits).
4. **Single-threaded table builds** — Notebooks 08–13 process JSON files sequentially in Python loops, then write to Spark. For 10,372 files this will be slow (~130s for current 3,518 at serving layer).
5. **No schema enforcement** — `overwriteSchema=true` means any extraction change silently breaks downstream consumers. No schema registry or compatibility checks.
6. **PII exposure** — Tax IDs and NPIs stored in cleartext in multiple tables with no column masking.
7. **Hardcoded paths** — All paths are absolute workspace paths to a single user's directory. Not relocatable without code changes.
8. **No monitoring** — No table freshness metrics, no row count alerting, no quality score trending over time.
9. **Prompt/schema coupling** — The V6 JSON schema is duplicated between notebooks 04 and 05. Schema changes require updating both.
10. **Coverage gap** — 57 providers are known-missing from rate tables (outpatient formula rates not extracted into rates_all). Documented but unfixed.

---

## 13. Detailed _config Notebook Analysis

### Widgets (Runtime Parameters)
* `run_mode`: dropdown ["sample", "full"] — controls whether all PDFs or a provider-folder sample
* `num_providers`: text (default "5") — sample size in provider-folder count
* `force_rebuild`: dropdown ["false", "true"] — override production gate

### Path Constants
* `BASE_DIR`: `/Workspace/Users/adixit01@blueshieldca.com/Contract_Codification_Pipeline`
* `DATA_DIR`: `/Volumes/prod_adb/default/ext-data-volume-stmlz/Health_Plan_Ops_Transformation/Provider_Contracts`
* `CHECKPOINT_DIR`: `{BASE_DIR}/checkpoints`
* `OUTPUT_DIR`: `{BASE_DIR}/json_extract`
* `CONSOLIDATED_DIR`: `{BASE_DIR}/json_consolidated`
* `BACKUP_DIR`: `{BASE_DIR}/json_extract_backup`
* `SCHEMA`: `dev_adb.raw`

### LLM Constants
* `LLM_MODEL`: `databricks-claude-sonnet-4-5`
* `REST_CONCURRENCY`: 3
* `TOKENS_PER_CHAR`: 0.3
* `PROMPT_TOKENS`: 700
* `CHUNK_SIZE`: 80,000
* `CHUNK_OVERLAP`: 2,000
* `RETRY_MAX`: 1
* `RETRY_BACKOFF`: 30
* `MAX_SINGLE_CHARS`: 270,000
* `PER_FILE_TIMEOUT`: 1,800
* `STEP3_TIME_BUDGET`: 24 * 3600 (24 hours)
* `MIN_SCORE_TEXT_LEN`: 5,000
* `MAX_OUTPUT_TOKENS`: 65,536

### Shared Helper Functions
* `clean_json_response(text)` — Extracts valid JSON from LLM response (removes fences, trailing commas, finds balanced braces)
* `safe_str(val, max_len)` — Safe string conversion with optional truncation
* `safe_float(val)` — Safe float conversion
* `sanitize_for_spark(df)` — Casts mixed-type object columns to string for Spark compatibility
* `write_base_table(rows, table_name)` — Writes list of dicts to Delta with OVERWRITE, handles NullType columns
* `add_col_if_missing(table_fqn, col_name, col_type)` — Safely adds column to existing Delta table

### REST API Auth
* Token: `dbutils.notebook.entry_point.getDbutils().notebook().getContext().apiToken().get()`
* Endpoint: `https://{WORKSPACE_URL}/serving-endpoints/{LLM_MODEL}/invocations`
* `call_llm(prompt_text, document_text, max_tokens, timeout)` — Returns (response_text, error_msg, usage_dict)

---

## 14. Table Schema Summary (All 35 Tables)

### Base Layer
| Table | Approx Rows | Key Columns |
|-------|------------|-------------|
| tbl_contract_rates_all | 68,418 | rate_category, service_category, rate_text, rate_numeric, status, program_normalized, amendment_order |
| tbl_contract_documents_master | 3,518 | source_filename, document_type, doc_category, amendment_order, effective_date_parsed |
| tbl_contract_financial_protections | 2,921 | protection_type, threshold_numeric, reimbursement_pct_normalized |
| tbl_contract_regional_factors | 7,864 | county, factor type columns, outpatient_surgical_factor_num |
| tbl_contract_clauses_terms | 47,107 | content_type (clause/section/definition), topic, content_text |
| tbl_contract_codes_events | 10,944 | content_type (medical_code/hac/never_event), code_value, category |
| tbl_contract_parties | 11,478 | party_role (health_plan/provider/signatory), party_name, tax_id, npi |
| tbl_contract_services_quality | 18,905 | service_type, delegated_activities, quality metrics |

### Dimension Tables
| Table | Rows | Purpose |
|-------|------|---------|
| dim_provider_canonical | 456 | 456 IDs → 278 facilities |
| dim_service_category | 26 | Service line reference |
| dim_network | ~185→12 | Network normalization |

### Genie Layer
| Table | Approx Rows | Purpose |
|-------|------------|--------|
| tbl_genie_provider_profile | 278 | Executive summary per facility |
| tbl_genie_rates_current | 12,278 | Deduplicated current rates |
| tbl_genie_contract_terms | 69,529 | Unified clauses/codes/parties |
| tbl_genie_amendment_timeline | 3,518 | Document chain per facility |

### Serving Layer
| Table | Approx Rows | Purpose |
|-------|------------|--------|
| tbl_serving_provider_summary | 278 | Executive profile |
| tbl_serving_rate_card | 3,113 | Inpatient per diem, 25 service lines |
| tbl_serving_capitation_card | 4,973 | PMPM rates |
| tbl_serving_case_rate_card | 1,632 | Bundled case rates |
| tbl_serving_outpatient_card | 3,954 | Outpatient rates |
| tbl_serving_stop_loss | 45 | One per facility |
| tbl_serving_amendment_history | 3,518 | Amendment timeline |
| tbl_serving_parties | 3,228 | Deduplicated parties |
| tbl_serving_regional_factors | 7,864 | Geographic factors |
| tbl_serving_services_coverage | 1,649 | DOFR + quality |

### Audit Tables
| Table | Purpose |
|-------|---------|
| tbl_extraction_quality_audit | File-level audit scores (MERGE) |
| tbl_extraction_quality_gaps | Per-gap detail (MERGE) |

---

## 15. Confidence Scoring Mechanisms

| Score/Flag | Source | Computation |
|-----------|--------|-------------|
| `v6_coverage_pct` | Validation (Notebook 06) | Weighted section completeness, doc-type-specific |
| `overall_quality_score` | LLM Audit (Notebook 14) | Claude assessment of extraction completeness |
| `date_verified` | Validation (Notebook 06) | Boolean: date string found in raw text |
| `rate_verified` | Validation (Notebook 06) | Boolean: rate value found in raw text (OCR-aware) |
| `is_valid_rate` | Enrichment (Notebook 11) | FALSE for negative/zero rates |
| `content_bucket` | Validation (Notebook 06) | Classification: full_contract, stub, cover_memo_only |

---

## END OF REPORT
