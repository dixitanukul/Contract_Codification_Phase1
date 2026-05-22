# Provider Contract Intelligence Platform — System Map (Phase 1)

*Reconstructed from the `Contract_Codification_Phase1` repository. Scope: structural/architectural map only. No business-logic deep dive, no redesign.*

The repository ships as **Databricks HTML notebook exports** (notebook source is embedded as base64-encoded JSON inside each `.html`). It contains **25 numbered notebooks** plus a shared `_config`, an `index`, a `README_tables` data dictionary, and eight `PHASE*.md` analysis documents. There are **no `.py`/`.sql`/`.ipynb` source files and no job/DLT definition files** — everything is notebook-resident.

---

## 1. Notebook Inventory

| # | Notebook | Subsystem | Primary Output |
|---|----------|-----------|----------------|
| — | `_config` | Shared runtime | constants, widgets, helpers, LLM REST auth (`%run` target for all) |
| 01 | `file_discovery` | Ingestion | `file_registry_v2.parquet`, seeds `step1_parsed_v2.parquet` |
| 02 | `ocr_extraction` | Ingestion | `step1_parsed_v2.parquet` (OCR text per PDF) |
| 03 | `metadata_parsing` | Extraction | `step2_with_metadata_v2.parquet` |
| 04 | `llm_prompts` | Extraction (lib) | prompt templates (`%run` into 05) |
| 05 | `llm_extraction` | Extraction | `json_extract/*.json` (one per PDF) via REST `call_llm` |
| 06 | `validation` | QA | enriches `json_extract/*.json` in place |
| 07 | `consolidation` | QA | `json_consolidated/` |
| 08 | `build_rates` | Base modeling | `tbl_contract_rates_all`, `_regional_factors`, `_financial_protections` |
| 09 | `build_terms` | Base modeling | `tbl_contract_clauses_terms`, `_codes_events`, `_parties`, `_services_quality` |
| 10 | `build_documents` | Base modeling | `tbl_contract_documents_master`, `dim_provider_canonical` |
| 11 | `enrich_tables` | Base modeling | adds/parses columns on base tables (in place) |
| 12 | `build_genie` | Genie layer | 4× `tbl_genie_*` (column-commented) |
| 13 | `build_serving` | Serving layer | 12× `tbl_serving_*` + `dim_service_category`, `dim_network` |
| 14 | `quality_audit` | QA | `tbl_extraction_quality_audit`, `tbl_extraction_quality_gaps` |
| 15 | `gap_filler` | QA | re-runs LLM on gaps → `tbl_extraction_gapfill_log` |
| 16 | `vector_search_indexing` | Retrieval infra | `*_vs_ready` tables + 2 Delta Sync vector indexes |
| 17 | `semantic_clause_search` | Intelligence | (interactive) vector search |
| 18 | `contract_comparison` | Intelligence | (interactive) pure SQL |
| 19 | `hybrid_query_router` | Intelligence | (interactive) LLM intent → SQL / VS — **integration hub** |
| 20 | `citation_enhanced_answers` | Intelligence | (interactive) VS + LLM |
| 21 | `amendment_impact_analysis` | Intelligence | (interactive) SQL |
| 22 | `clause_similarity_matrix` | Intelligence | (interactive) VS |
| 23 | `grounded_legal_qa` | Intelligence | (interactive) VS + LLM (RAG) |
| 24 | `contract_renewal_intelligence` | Intelligence | (interactive) pure SQL |
| 25 | `negotiation_intelligence` | Intelligence | (interactive) LLM + SQL |

---

## 2. Execution Order & Orchestration

**There is no orchestration layer in code** — no Databricks Jobs/Workflows, no Delta Live Tables, no `dbutils.notebook.run`. Execution order is implied entirely by the **filename numbering convention (01 → 25)** and run manually. The only code-level coupling is:

- `%run ./_config` — every pipeline notebook (loads shared constants, helpers, LLM auth).
- `%run ./04_llm_prompts` — invoked inside `05_llm_extraction`.

Each retrieval notebook that needs the Vector Search SDK runs `%pip install` + `restartPython()` **before** `%run ./_config` (a documented ordering constraint, because `restartPython()` inside a `%run` target hangs).

```
_config  ──(%run)── every notebook
04_llm_prompts ──(%run)── 05_llm_extraction

01 → 02 → 03 → 05(+04) → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13 → 14 → 15 → 16 → {17..25}
        (ingest/OCR)     (extract)   (consolidate)   (base Delta)   (genie)(serving)(QA)  (vector) (intelligence apps)
```

---

## 3. End-to-End Lineage

```
prod_adb volume  /Volumes/.../Provider_Contracts/*.pdf   (~10.3K PDFs, 27 GB, 597 providers)
        │
   [01] os.walk → file_registry_v2.parquet
        │
   [02] OCR ─────────────► checkpoints/step1_parsed_v2.parquet ─────────────────────────┐
        │                                                                                │
   [03] metadata ────────► checkpoints/step2_with_metadata_v2.parquet                    │
        │                                                                                │
   [05] LLM extract (REST, prompts from 04) ─► json_extract/*.json                       │
        │                                                                                │
   [06] validate in-place ─► json_extract/*.json (enriched)                              │
        │                                                                                │
   [07] consolidate ──────► json_consolidated/                                           │
        │                                                                                │
        ├──► [08/09/10] BASE DELTA LAYER (dev_adb.raw)                                    │
        │        tbl_contract_rates_all, _regional_factors, _financial_protections,       │
        │        _clauses_terms, _codes_events, _parties, _services_quality,              │
        │        tbl_contract_documents_master, dim_provider_canonical                    │
        │              │                                                                  │
        │        [11] enrich (date parsing, added columns; in place)                      │
        │              │                                                                  │
        │              ├──► [12] GENIE LAYER: tbl_genie_provider_profile, _rates_current, │
        │              │            _contract_terms, _amendment_timeline                  │
        │              │                 │                                                │
        │              │                 └──► [16] structured index source                │
        │              │                                                                  │
        │              └──► [13] SERVING LAYER: 12× tbl_serving_* + dim_service_category, │
        │                          dim_network                                           │
        │                                                                                │
        └──► [14] quality_audit (reads json_extract) → _quality_audit / _quality_gaps     │
                     │                                                                    │
               [15] gap_filler (re-LLM) → _gapfill_log                                    │
                                                                                          │
   [16] VECTOR SEARCH (endpoint: contract_intelligence_vs_endpoint, model: gte-large-en) │
        • structured: tbl_genie_contract_terms → tbl_contract_terms_vs_ready → tbl_contract_terms_vs_index
        • full-text:  step1_parsed_v2.parquet ◄──────────────────────────────────────────┘ (chunked)
                       → tbl_contract_fulltext_vs_ready → tbl_contract_fulltext_vs_index
        │
   [17..25] INTELLIGENCE APPS  (read genie/serving tables + 2 vector indexes; no persisted output)
```

**Two persistence regimes.** Notebooks 01–07 persist to **file checkpoints** (parquet + per-PDF JSON under `/Workspace/.../Contract_Codification_Pipeline`). The handoff to **Delta tables** (`dev_adb.raw`) happens at notebook 08. Note the full-text vector index is sourced from the **file checkpoint** (`step1_parsed_v2.parquet`), bypassing the governed Delta layer — the only consumer that does so.

---

## 4. Delta Table Lineage (layered model)

```
BASE (08–11)                         GENIE (12)                    SERVING (13)
tbl_contract_rates_all          ┐    tbl_genie_provider_profile    tbl_serving_provider_summary
tbl_contract_regional_factors   │    tbl_genie_rates_current       tbl_serving_rate_card
tbl_contract_financial_prot.    ├──► tbl_genie_contract_terms ─┐   tbl_serving_capitation_card
tbl_contract_clauses_terms      │    tbl_genie_amendment_timeline  tbl_serving_case_rate_card
tbl_contract_codes_events       │                              │   tbl_serving_outpatient_card
tbl_contract_parties            │                              │   tbl_serving_stop_loss
tbl_contract_services_quality   │                              │   tbl_serving_amendment_history
tbl_contract_documents_master   │                              │   tbl_serving_parties
dim_provider_canonical (join key)┘                             │   tbl_serving_regional_factors
                                                               │   tbl_serving_services_coverage
QA (14–15)                                                     │   + dim_service_category, dim_network
tbl_extraction_quality_audit                                   │
tbl_extraction_quality_gaps                          VECTOR (16)│
tbl_extraction_gapfill_log         tbl_contract_terms_vs_ready ◄┘ → tbl_contract_terms_vs_index
                                   tbl_contract_fulltext_vs_ready  → tbl_contract_fulltext_vs_index
```

`dim_provider_canonical` is the universal join key, resolving multiple `provider_id`s → distinct facilities. Genie and serving tables pre-join through it. (README documents ~35 tables; row counts there are documentation, not verified against a live catalog in Phase 1.)

---

## 5. Dependency Summary by Type

- **Vector index dependencies:** both indexes live on one endpoint, `contract_intelligence_vs_endpoint`, using `databricks-gte-large-en` (1024-dim), Delta Sync with Change Data Feed. Structured ← `tbl_genie_contract_terms`; full-text ← OCR checkpoint parquet. Consumed by notebooks 17, 19, 20, 22, 23.
- **"Serving layer" dependencies:** the term refers to **12 pre-joined analytical Delta tables**, not a model/feature-serving endpoint. No MLflow model logging or serving endpoint is registered anywhere in the repo.
- **Genie dependencies:** four `tbl_genie_*` tables with full column comments. The **Genie space is not created in code** — it is configured manually in the Databricks UI against these tables.
- **LLM dependencies:** a single direct-REST pattern (`call_llm` in `_config`) to `databricks-claude-sonnet-4-5`. Used by 05 (extraction), 14/15 (audit/gap-fill), and intelligence notebooks 19, 20, 23, 25 (`ai_query()` is deliberately avoided to dodge Spark batch timeouts).

---

## 6. Core Subsystems

1. **Shared runtime** — `_config` (+ `04_llm_prompts` as a prompt library).
2. **Ingestion & OCR** — 01–02 (file discovery + OCR to parquet checkpoints).
3. **LLM extraction** — 03–05 (metadata → per-PDF JSON via REST).
4. **Validation & consolidation** — 06–07 (JSON QA, in-place enrichment, consolidation).
5. **Base Delta modeling** — 08–11 (normalized contract tables + canonical provider dimension).
6. **Genie semantic-table layer** — 12 (NL-optimized, commented tables).
7. **Analytical serving layer** — 13 (deduped, type-separated reporting tables).
8. **Quality assurance & gap-fill** — 14–15 (audit, gaps, LLM re-extraction).
9. **Vector search indexing** — 16 (two Delta Sync indexes on one endpoint).
10. **Retrieval & intelligence apps** — 17–25 (interactive consumers; semantic search, hybrid routing, RAG QA, comparison, amendment/renewal/negotiation analytics).

---

## 7. Implemented vs. Planned / Brittle Paths (structural flags only)

- **Implemented in code:** the full 01–16 build pipeline, both vector indexes, and all 17–25 query notebooks.
- **Implicit orchestration (brittle):** correct end-to-end runs depend on a human executing notebooks in numeric order; re-run/freshness dependencies (e.g., a base-table change requiring 12 → 16 re-runs) are not enforced anywhere.
- **Lineage leak:** the full-text index reads a file checkpoint rather than a Delta table, so it is outside Unity Catalog lineage/governance and can drift from the Delta corpus.
- **Naming overloads to watch in later phases:** "serving" = analytical tables (not model serving); "Genie layer" = tables only (space is manual).
- **Environment:** source PDFs are read from a **prod** volume while all tables are written to a **dev** schema (`dev_adb.raw`).
- **Intelligence notebooks are interactive/demo-shaped:** no persisted outputs; several rely on LLM calls with hardcoded fallback branches (e.g., the 19 intent classifier), which is a likely brittle demo path to scrutinize in Phase 2+.

---

*Persistent architectural understanding for this review has been saved to memory; subsequent phases can build on this map without re-scanning unchanged areas.*
