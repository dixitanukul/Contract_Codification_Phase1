# PHASE 6 — Enterprise Evaluation, Testing, Benchmarking & QA Framework
## Contract Intelligence Platform — Complete Evaluation Architecture
**Date:** May 19, 2026  
**Role:** Principal AI Evaluation Architect / Enterprise GenAI Reliability Engineer / Legal AI Benchmarking Specialist  
**Basis:** Live data inspection of 3,509 JSON extractions, 35 Delta tables, audit/gaps tables, v6 coverage scores, rate verification metrics

---

## 1. Executive Summary

This document defines the complete evaluation, testing, and quality assurance framework for the Contract Intelligence Platform. The framework ensures that every extracted rate, clause, date, and legal term can be measured for accuracy, that regressions are caught before reaching production, and that enterprise trust is quantifiable rather than assumed.

The current system has **foundational evaluation capabilities** (v6 coverage scoring, date/rate source verification, LLM-based quality audit with gap detection) but lacks the rigor required for enterprise legal reliance: no golden datasets, no field-level precision/recall measurement, no retrieval quality metrics, no regression detection, and no continuous production monitoring.

The proposed framework introduces **25 distinct evaluation dimensions** organized into five operational tiers:

1. **Extraction Evaluation** — measuring what the LLM produces against ground truth
2. **Data Quality Evaluation** — measuring what reaches Delta tables
3. **Retrieval & QA Evaluation** — measuring answer quality and grounding
4. **Regression & Drift Evaluation** — detecting silent degradation
5. **Enterprise Trust Evaluation** — computing composite trustworthiness scores

**Key finding from current-state analysis:** The existing evaluation mechanisms (v6_coverage_pct, rate verification, LLM audit) measure **presence** and **source existence**, not **correctness** or **attribution accuracy**. A rate can be verified as existing in source text (98.2% verification rate) while being attributed to the wrong service category — an error invisible to all current checks.

---

## 2. Current Evaluation Gap Analysis

### 2.1 What Currently Exists (Measured from Live Data)

| Mechanism | What It Measures | Coverage | Reliability |
|-----------|-----------------|----------|-------------|
| `v6_coverage_pct` | Section presence (14 categories scored 0/1) | 3,509 files | MEDIUM — measures presence, not correctness |
| `v6_coverage_scores` | Per-section binary presence | 3,509 files | MEDIUM — same caveat |
| Rate source verification | Whether `rate_numeric` value exists anywhere in OCR text | 98.2% verified | LOW — verifies value existence, not attribution |
| Date source verification | Whether date string exists in OCR text | 4,800 dates verified | MEDIUM — string match is reasonable for dates |
| `tbl_extraction_quality_audit` | LLM self-assessment of extraction quality | 142 documents | LOW — circular (same model), all have empty error field |
| `tbl_extraction_quality_gaps` | LLM-identified missing values, truncations, hallucinations | 6,877 gap records | LOW-MEDIUM — identifies types but cannot verify |
| `tbl_extraction_gapfill_log` | Gap-filling success tracking | 129 files processed | HIGH — operational log |
| `is_valid_rate` flag | Negative/zero rate detection | 53 invalid rates found | HIGH — deterministic rule |
| `validation_issues` in JSON | Missing required fields at extraction time | Per-file list | HIGH — deterministic presence checks |

### 2.2 Current Score Distribution (Measured)

| Metric | Value |
|--------|-------|
| Mean v6_coverage_pct | 49.2% |
| Median v6_coverage_pct | 50.0% |
| Files with <25% coverage | 318 (9.1%) |
| Files with >75% coverage | 648 (18.5%) |
| Rate verification rate | 98.2% (33,657 / 34,285) |
| Files with >=1 verified date | 3,156 / 3,519 (89.7%) |
| HIGH severity gaps identified | 981 total (593 missing_rows, 339 missing_value, 23 empty_field, 20 misclassification, 6 hallucination) |
| Hallucinations explicitly flagged | 6 total (in gap table) |

### 2.3 Critical Evaluation Gaps

| Gap | Impact | Priority |
|-----|--------|----------|
| **No golden dataset** — zero human-annotated ground truth | Cannot compute precision, recall, F1 for any extraction task | CRITICAL |
| **No rate-service attribution validation** — only checks rate VALUE exists | 15% misattribution completely invisible | CRITICAL |
| **No clause completeness validation** — cannot tell if clause is truncated | Truncated legal text appears valid | CRITICAL |
| **No retrieval quality measurement** — no queries, no relevance labels | Cannot assess semantic search if implemented | HIGH |
| **No regression detection** — no before/after comparison on prompt/model changes | Silent degradation on model endpoint updates | HIGH |
| **No cross-document consistency checks** — same rate from 2 docs not compared | Conflicts go undetected | HIGH |
| **No production drift monitoring** — no row count, NULL rate, or distribution tracking | Degradation detected manually after days/weeks | HIGH |
| **No human-review agreement measurement** — no human labels to compare against | Cannot calibrate system confidence | MEDIUM |
| **No embedding quality metrics** — no vector search yet | N/A until retrieval built | FUTURE |
| **No end-to-end trust score** — no composite metric for enterprise reporting | Trust is qualitative, not quantifiable | MEDIUM |

### 2.4 Where Hallucinations Are Currently Undetectable

| Hallucination Type | Why Undetectable |
|--------------------|------------------|
| Rate attributed to wrong service | Rate value verified in source, but column assignment not checked |
| Effective date fabricated within plausible range | Only string-match verification; dates in 1990-2027 range pass |
| Program defaulted vs extracted | No field-level provenance marker |
| Clause truncated mid-paragraph | Partial text still valid syntactically |
| Amendment impact summary invented | LLM-generated text with no source verification |
| Contract overview hallucinated | Overview not string-matched against source |
| Provider name substituted (parent for subsidiary) | Filename-derived provider_id takes precedence regardless |

### 2.5 Where Regressions Can Silently Occur

| Regression Vector | Trigger | Detection Mechanism |
|-------------------|---------|--------------------|
| Model endpoint updated | Databricks updates Claude Sonnet 4.5 | NONE — no version check |
| Prompt change | Developer modifies extraction prompt | NONE — no prompt versioning |
| Schema format change in LLM output | Model returns dates in new format | NONE — parsing silently fails to NULL |
| OCR quality degradation | PDF rendering library update | NONE — no OCR quality tracking |
| Chunking logic change | Overlap/size parameter modified | NONE — no extraction comparison |

---

## 3. Future-State Evaluation Architecture

### 3.1 Architecture Overview

```
+------------------------------------------------------------------------+
|                    EVALUATION & QA ARCHITECTURE                          |
+------------------------------------------------------------------------+
|                                                                          |
|  +---------------------+     +----------------------+                    |
|  |  GOLDEN DATASETS    |     |  EVALUATION ENGINE   |                    |
|  |  * Annotated docs   |---->|  * Extraction eval   |                    |
|  |  * Labeled clauses  |     |  * Retrieval eval    |                    |
|  |  * Verified rates   |     |  * QA eval           |                    |
|  |  * Review decisions |     |  * Trust scoring     |                    |
|  +---------------------+     +----------+-----------+                    |
|                                         |                                |
|  +---------------------+     +----------v-----------+                    |
|  |  REGRESSION SUITE   |     |  EVALUATION RESULTS  |                    |
|  |  * Prompt tests     |---->|  Delta tables        |                    |
|  |  * Model tests      |     |  * eval_extraction   |                    |
|  |  * Schema tests     |     |  * eval_retrieval    |                    |
|  |  * Drift tests      |     |  * eval_trust_scores |                    |
|  +---------------------+     |  * eval_regression   |                    |
|                              +----------+-----------+                    |
|  +---------------------+               |                                 |
|  |  CONTINUOUS MONITOR |               |                                 |
|  |  * Data quality SLA |<--------------+                                 |
|  |  * Drift detection  |                                                 |
|  |  * Freshness checks |     +----------------------+                    |
|  |  * Alert triggers   |---->|  DASHBOARDS & ALERTS |                    |
|  +---------------------+     +----------------------+                    |
|                                                                          |
|  +---------------------+     +----------------------+                    |
|  |  HUMAN REVIEW LOOP  |     |  MLFLOW TRACKING     |                    |
|  |  * Review queue     |---->|  * Experiment runs   |                    |
|  |  * IAA scoring      |     |  * Model comparison  |                    |
|  |  * Calibration      |     |  * Prompt comparison  |                    |
|  +---------------------+     +----------------------+                    |
|                                                                          |
+------------------------------------------------------------------------+
```

### 3.2 Evaluation Tiers

| Tier | Purpose | Frequency | Trigger |
|------|---------|-----------|--------|
| **T1: Unit evaluation** | Per-field extraction correctness | Every extraction | Automated |
| **T2: Benchmark evaluation** | Golden dataset precision/recall | Pre-deployment + weekly | Prompt/model change |
| **T3: Regression evaluation** | Before/after comparison | Every code/model change | CI/CD gate |
| **T4: Production monitoring** | Drift, SLA, anomaly detection | Continuous (hourly/daily) | Scheduled |
| **T5: Human calibration** | Agreement scoring, trust calibration | Monthly + on-demand | Scheduled + review |

---

## 4. Golden Dataset Strategy

### 4.1 What Golden Datasets Must Contain

For provider contract evaluation, the golden dataset is NOT a generic labeled corpus. It must be **contract-specific, multi-layer, and legally verified**.

### 4.2 Golden Dataset Structure

#### Layer 1: Document-Level Ground Truth

| Field | Description | Annotation Method |
|-------|-------------|-------------------|
| document_type | Correct doc classification | Human label |
| provider_name | Correct canonical provider | Human verified |
| effective_date | Correct date from execution page | Human verified |
| amendment_order | Correct position in chain | Human verified |
| page_count | Correct page count | Automated |
| has_rate_schedule | Whether document contains rates | Human label |
| has_termination_clause | Whether doc has term clause | Human label |

#### Layer 2: Rate Table Ground Truth

| Field | Description | Annotation Method |
|-------|-------------|-------------------|
| rate_numeric | Correct dollar amount | Human transcription |
| service_category | Correct service line assignment | Human label |
| rate_category | Correct rate type | Human label |
| program | Correct program assignment | Human label |
| network | Correct network tier | Human label |
| page_number | Page where rate appears | Human label |
| table_row_id | Which table row this came from | Human label |

#### Layer 3: Clause Ground Truth

| Field | Description | Annotation Method |
|-------|-------------|-------------------|
| clause_text | Complete verbatim text | Human transcription |
| clause_type | Correct topic classification | Human label |
| clause_start_page | Page where clause begins | Human label |
| clause_end_page | Page where clause ends | Human label |
| is_complete | Whether full clause captured | Human verification |
| obligations | Key obligations identified | Human label |

#### Layer 4: Amendment Lineage Ground Truth

| Field | Description | Annotation Method |
|-------|-------------|-------------------|
| supersedes_document | Which prior document is superseded | Human label |
| supersedes_rates | Which specific rates are replaced | Human label |
| supersedes_clauses | Which specific clauses are replaced | Human label |
| effective_date_change | Whether effective date changed | Human label |

### 4.3 Golden Dataset Sizing

| Segment | Documents | Purpose |
|---------|-----------|--------|
| Simple base agreements (short, single exhibit) | 10 | Baseline extraction accuracy |
| Complex base agreements (multi-exhibit, 50+ pages) | 10 | Scale handling |
| Single amendments (rate change only) | 10 | Amendment impact detection |
| Multi-amendment chains (3+ amendments) | 5 chains (15-20 docs) | Supersession testing |
| Cover memos | 5 | Lightweight extraction |
| Settlements | 5 | Settlement-specific schema |
| Fee schedules with complex tables | 10 | Table extraction accuracy |
| Cross-page tables | 5 | Page continuation handling |
| Documents with formulas/percentages (no rate_numeric) | 5 | Non-numeric rate handling |
| **TOTAL** | **~75 documents** | Full evaluation coverage |

### 4.4 Annotation Protocol

1. **Source**: select documents stratified by provider size, document type, complexity
2. **Primary annotator**: contract analyst familiar with healthcare reimbursement
3. **Secondary annotator**: second analyst for inter-annotator agreement
4. **Adjudication**: legal/reimbursement SME resolves disagreements
5. **Storage**: annotated ground truth stored in Delta table `eval_golden_documents` with version tracking
6. **Refresh**: add 5 new documents per quarter; retire documents if contracts are re-negotiated

### 4.5 Golden Dataset Tables

```sql
-- Ground truth for document-level facts
eval_golden_documents (
  golden_id STRING,
  source_filename STRING,
  provider_id STRING,
  annotated_document_type STRING,
  annotated_effective_date DATE,
  annotated_provider_name STRING,
  annotated_amendment_order INT,
  annotated_has_rates BOOLEAN,
  annotated_has_clauses BOOLEAN,
  annotator_id STRING,
  annotation_date TIMESTAMP,
  annotation_version INT
)

-- Ground truth for individual rate rows
eval_golden_rates (
  golden_id STRING,
  source_filename STRING,
  rate_numeric DOUBLE,
  service_category STRING,
  rate_category STRING,
  program STRING,
  network STRING,
  page_number INT,
  table_id STRING,
  row_position INT,
  annotator_id STRING,
  annotation_date TIMESTAMP
)

-- Ground truth for clauses
eval_golden_clauses (
  golden_id STRING,
  source_filename STRING,
  clause_text STRING,
  clause_type STRING,
  page_start INT,
  page_end INT,
  is_complete BOOLEAN,
  word_count INT,
  annotator_id STRING,
  annotation_date TIMESTAMP
)

-- Ground truth for amendment linkages
eval_golden_amendments (
  golden_id STRING,
  amendment_filename STRING,
  base_filename STRING,
  supersedes_rates ARRAY<STRING>,
  supersedes_clauses ARRAY<STRING>,
  annotator_id STRING,
  annotation_date TIMESTAMP
)
```

---

## 5. OCR Quality Evaluation Framework

### 5.1 Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Character accuracy | % chars correctly recognized vs source | >99% (digital-born) |
| Word accuracy | % words matching source (Levenshtein < 2) | >97% |
| Layout preservation | % of reading-order-correct paragraphs | >95% |
| Table detection recall | % of tables in document correctly identified | >90% |
| Table detection precision | % of detected tables that are actual tables | >95% |
| Cross-page table continuation | % of multi-page tables correctly merged | >80% |
| OCR tier assignment accuracy | % correct routing (digital vs scanned) | >98% |

### 5.2 Automated OCR Quality Checks

Checks that run on every document without ground truth:

* `min_text_length`: len(text) > 100 (not empty)
* `char_density`: len(text) / page_count > 200 (reasonable content per page)
* `repeated_garbage`: no OCR repetition loops
* `encoding_valid`: text encodes cleanly to UTF-8
* `digit_ratio`: digits < 40% of text (not corrupted)
* `whitespace_ratio`: whitespace < 60% (not mostly blank)

---

## 6. Table Extraction Evaluation Framework

### 6.1 The Critical Challenge

Fee schedule tables contain the highest-value data AND are where extraction errors have the greatest financial impact. Table evaluation requires cell-level precision.

### 6.2 Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Table detection recall | Tables found / tables in document | >95% |
| Row extraction recall | Rows extracted / rows in source table | >95% |
| Row extraction precision | Correct rows / rows extracted | >90% |
| Cell value accuracy | Correct cell values / total cells extracted | >95% |
| Column assignment accuracy | Cells in correct column / total cells | >90% |
| Rate-service attribution accuracy | Rate assigned to correct service / total rates | >90% (current est: ~85%) |
| Header detection accuracy | Correct header row identified / tables | >95% |
| Cross-page table merge accuracy | Correctly continued tables / multi-page tables | >80% |

### 6.3 Rate Attribution Evaluation Protocol

The MOST IMPORTANT extraction evaluation for this domain:

```
For each golden rate row:
  1. Find matching extracted rate by (source_filename + rate_numeric + page_number)
  2. Compare service_category assignment:
     - EXACT_MATCH: extracted service == golden service
     - PARTIAL_MATCH: extracted service contains golden service keywords
     - MISMATCH: completely wrong service assignment
     - MISSING: rate exists in golden but not in extraction
     - SPURIOUS: rate exists in extraction but not in golden
  3. Compare program assignment
  4. Compare network assignment
  5. Compute attribution F1:
     - Precision = correct attributions / total extracted
     - Recall = correct attributions / total in golden
```

### 6.4 Table Reconstruction Score

Composite metric:

```
Table_Reconstruction_Score = 
  0.30 x row_recall +
  0.30 x cell_accuracy +
  0.20 x column_assignment_accuracy +
  0.10 x header_detection +
  0.10 x cross_page_merge_accuracy
```

---

## 7. Clause Extraction Evaluation Framework

### 7.1 Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Clause detection recall | Clauses found / clauses in document | >85% |
| Clause detection precision | Valid clauses / clauses extracted | >90% |
| Clause completeness score | % of clause text captured (token overlap) | >95% |
| Clause type accuracy | Correct topic / total classified | >85% |
| Clause boundary accuracy | Start/end within 1 sentence of golden boundary | >90% |
| Full clause text fidelity | Exact text match rate (normalized whitespace) | >90% |

### 7.2 Clause Completeness Evaluation

The most critical clause metric is **completeness** — a truncated clause missing its exceptions or carve-outs is legally dangerous:

```python
def clause_completeness_score(extracted_text, golden_text):
    """Measures what fraction of the golden clause was captured.
    Extra penalty for missing the LAST 20% (often contains exceptions)."""
    extracted_tokens = set(extracted_text.lower().split())
    golden_tokens = set(golden_text.lower().split())
    
    if not golden_tokens:
        return 1.0
    
    recall = len(extracted_tokens & golden_tokens) / len(golden_tokens)
    
    # Penalty for missing the LAST 20% of clause
    golden_last_20pct = golden_text[int(len(golden_text) * 0.8):]
    last_20_tokens = set(golden_last_20pct.lower().split())
    tail_recall = len(extracted_tokens & last_20_tokens) / max(len(last_20_tokens), 1)
    
    return 0.7 * recall + 0.3 * tail_recall
```

### 7.3 Clause Type Classification Evaluation

| Clause Type | Expected Frequency | Evaluation |
|-------------|-------------------|------------|
| termination | ~90% of base agreements | Label vs golden |
| confidentiality | ~85% of base agreements | Label vs golden |
| dispute_resolution | ~70% of base agreements | Label vs golden |
| audit_rights | ~60% of base agreements | Label vs golden |
| indemnification | ~50% of base agreements | Label vs golden |
| non_compete | ~30% of base agreements | Label vs golden |
| assignment | ~40% of base agreements | Label vs golden |

---

## 8. Retrieval & RAG Evaluation Framework

### 8.1 Retrieval Evaluation Query Set

Benchmark with 100+ queries across categories:

| Query Type | Example | Expected Path |
|-----------|---------|---------------|
| Provider-specific rate lookup | "What is Kaiser's ICU per diem?" | SQL -> specific rate row |
| Cross-provider comparison | "Compare stop-loss thresholds" | SQL -> aggregation |
| Clause semantic search | "Find arbitration clauses" | Vector -> clause rows |
| Legal concept exploration | "Which contracts have most-favored-nation clauses?" | Vector -> concept match |
| Amendment impact | "What changed in Garfield's 2024 amendment?" | Hybrid -> amendment + rates |
| Historical state | "What was the rate in 2021?" | SQL -> historical state |
| Unsupported question | "What is the CEO's opinion on rates?" | Abstention expected |

### 8.2 Retrieval Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Precision@5 | Relevant docs in top 5 / 5 | >0.80 |
| Precision@10 | Relevant docs in top 10 / 10 | >0.70 |
| Recall@10 | Relevant docs in top 10 / total relevant | >0.90 |
| MRR (Mean Reciprocal Rank) | 1/rank of first relevant result | >0.85 |
| NDCG@10 | Normalized discounted cumulative gain | >0.80 |
| Provider filter precision | Correct provider scoping / total queries | >0.95 |
| Current-state filter precision | Only current items returned / total | >0.95 |
| Abstention recall | Correctly abstained / unsupported queries | >0.90 |

### 8.3 RAG Grounding Evaluation

| Metric | Definition | Target |
|--------|-----------|--------|
| Groundedness score | % of answer claims supported by retrieved context | >0.95 |
| Citation precision | Correct citations / total citations | >0.90 |
| Citation recall | Claims with correct citation / total claims | >0.85 |
| Unsupported claim rate | Claims not grounded in any retrieved context | <0.05 |
| Hallucination rate | Claims contradicted by retrieved context | <0.02 |
| Faithfulness score | Semantic similarity between answer and source | >0.90 |
| Abstention correctness | Correct refusal for unanswerable questions | >0.90 |

### 8.4 RAG Evaluation Protocol

For each evaluation query:

1. **Route classification**: was query sent to correct lane (SQL/vector/hybrid)?
2. **Retrieval quality**: are retrieved chunks relevant? (human or LLM-judge label)
3. **Answer groundedness**: does every claim have supporting evidence?
4. **Citation correctness**: does each citation point to actual supporting text?
5. **Answer correctness**: does answer match golden answer? (human verified)
6. **Abstention behavior**: did system refuse when it should have?

---

## 9. Hallucination Detection Framework

### 9.1 Hallucination Taxonomy for Provider Contracts

| Type | Description | Detection Method | Automation |
|------|-------------|-----------------|------------|
| **Intrinsic fabrication** | LLM invents a value not in source | Source text search | Automatable |
| **Attribution error** | Value exists but assigned to wrong field | Cross-field validation | Semi-automatable |
| **Temporal fabrication** | Date is plausible but not in source | Source text search + range check | Automatable |
| **Entity substitution** | Parent company name used instead of facility | Entity resolution check | Semi-automatable |
| **Completion hallucination** | LLM continues/extends incomplete text | Length comparison with source | Automatable |
| **Summarization invention** | Summary contains facts not in source | Entailment check | LLM-judge |
| **Cross-document contamination** | Facts from another contract appear | Document isolation check | Automatable |
| **Default masquerading** | System default appears as extracted value | Provenance tracking | Automatable |

### 9.2 Automated Hallucination Detection Pipeline

```
For each extracted field:
  1. SOURCE_CHECK: Is the exact value present in the source OCR text?
     - Numbers: fuzzy match within formatting tolerance ($1,234 = 1234)
     - Dates: multiple format matching (Jan 1 2023 = 2023-01-01 = 1/1/23)
     - Text: substring match with normalization
  
  2. ATTRIBUTION_CHECK: Is the value in the correct structural context?
     - Rate next to correct service heading?
     - Date in correct section (effective date in header, not body)?
  
  3. PLAUSIBILITY_CHECK: Is the value within expected range?
     - Rates: $0-$200K (flag if >$50K for non-transplant)
     - Dates: 1995-2028 (flag if outside range)
     - Percentages: 0-300% (flag if >200%)
  
  4. CONSISTENCY_CHECK: Does value conflict with other extractions?
     - Same provider, same service: rates should be similar across amendments
     - Same contract: effective date should not precede execution date
  
  5. PROVENANCE_LABEL: Assign hallucination risk level
     - GREEN: passes all checks
     - YELLOW: passes source check but fails attribution or plausibility
     - RED: fails source check OR creates consistency conflict
     - UNKNOWN: cannot verify (source text unavailable or ambiguous)
```

### 9.3 Hallucination Metrics

| Metric | Formula | Target |
|--------|---------|--------|
| Source-verifiable rate | rates with source match / total rates | >0.95 |
| Source-verifiable date | dates with source match / total dates | >0.90 |
| Attribution-verifiable rate | rates with correct context / total verified | >0.90 |
| Plausibility pass rate | values in expected range / total values | >0.99 |
| Cross-doc consistency rate | non-conflicting values / total comparable | >0.95 |
| **Composite hallucination score** | weighted average of all checks | >0.92 |

### 9.4 LLM-as-Judge for Hallucination Detection

Use a DIFFERENT model (not the extraction model) as judge:

```
Judge prompt structure:
  Given: 
    - Source PDF text (relevant pages)
    - Extracted field name and value
    - Surrounding context from extraction
  
  Determine:
    1. Is this value present in the source? (YES/NO/AMBIGUOUS)
    2. Is it attributed to the correct entity/service? (YES/NO/AMBIGUOUS)
    3. Is it complete or truncated? (COMPLETE/TRUNCATED/UNKNOWN)
    4. Confidence in assessment: (HIGH/MEDIUM/LOW)
    
  Constraints:
    - Judge must cite specific page/paragraph in source
    - Judge must explain reasoning
    - If ambiguous, must state what additional info is needed
```

---

## 10. Legal Q&A Evaluation Framework

### 10.1 Question Categories

| Category | Example | Expected Answer Type |
|----------|---------|---------------------|
| Factual lookup | "What is Garfield's ICU rate?" | Specific value + citation |
| Existence check | "Does Sutter have stop-loss?" | Boolean + citation |
| Comparison | "Compare Kaiser vs Sutter ICU rates" | Table + citations |
| Temporal | "What was the rate before 2024 amendment?" | Historical value + reference |
| Legal interpretation | "What are the termination conditions?" | Clause text + analysis |
| Aggregation | "Average inpatient rate across all providers" | Computed value |
| Unsupported | "Will rates increase next year?" | Abstention |

### 10.2 Answer Quality Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| Factual accuracy | Correct facts / total facts in answer | >0.95 |
| Completeness | Relevant facts included / total relevant facts | >0.85 |
| Citation correctness | Valid citations / total citations | >0.90 |
| Currentness | Answer uses current (not superseded) data | >0.95 |
| Abstention precision | Correct abstentions / total abstentions | >0.90 |
| Abstention recall | Correct abstentions / unanswerable questions | >0.85 |
| Legal safety | No legally dangerous incorrect claims | 100% |

### 10.3 Legal Safety Evaluation

| Dangerous Answer Type | Detection Method | Required Action |
|----------------------|------------------|-----------------|
| Wrong termination date | Compare vs golden | Hard block if wrong |
| Wrong settlement amount | Compare vs golden | Hard block if wrong |
| Wrong stop-loss threshold | Compare vs golden | Human review required |
| Superseded rate presented as current | Check status field | Hard block |
| Incomplete clause missing exception | Clause completeness check | Warning + full source |

---

## 11. Regression Testing Framework

### 11.1 Regression Test Suite Structure

| Suite | What Changes | What Stays Fixed | Tests |
|-------|-------------|-----------------|-------|
| **Prompt regression** | Extraction prompt | Model, inputs, schema | Re-extract golden docs, compare field-by-field |
| **Model regression** | Model version/endpoint | Prompt, inputs, schema | Re-extract golden docs, compare outputs |
| **Schema regression** | Output schema | Model, prompt | Verify all required fields populated |
| **OCR regression** | OCR library/method | Everything else | Compare OCR text output for golden docs |
| **Chunking regression** | Chunk params | Model, prompt | Compare chunk boundaries for golden docs |
| **Pipeline regression** | Any pipeline code | Golden inputs | End-to-end table comparison |

### 11.2 Regression Detection Protocol

```
For each regression test run:
  1. Extract the golden document set with new configuration
  2. Compare field-by-field against baseline extraction:
     - IDENTICAL: same value
     - IMPROVED: new value matches golden, old didn't
     - DEGRADED: old value matched golden, new doesn't
     - CHANGED: both differ from golden (neutral)
  3. Compute regression score:
     - If DEGRADED count > 0 for any HIGH-RISK field: FAIL
     - If DEGRADED count > 5% of fields: WARN
     - If IMPROVED count > DEGRADED count: PASS (net improvement)
  4. Generate regression report with specific field-level diffs
```

### 11.3 Critical Regression Gates

Must PASS before any model/prompt change is deployed:

| Gate | Condition | Action if Fails |
|------|-----------|----------------|
| Rate extraction precision >= baseline | precision(new) >= precision(old) - 0.02 | Block deployment |
| Date extraction recall >= baseline | recall(new) >= recall(old) - 0.02 | Block deployment |
| Zero increase in hallucinated dates | halluc_dates(new) <= halluc_dates(old) | Block deployment |
| Clause completeness >= baseline | completeness(new) >= completeness(old) - 0.05 | Block deployment |
| No new NULL explosions | null_rate(new) <= null_rate(old) + 0.01 | Block deployment |
| Service attribution >= baseline | attribution_acc(new) >= attribution_acc(old) - 0.03 | Warn + review |

### 11.4 Model Version Regression Testing

When Databricks updates `databricks-claude-sonnet-4-5`:

1. Detect version change (check model metadata on first invocation)
2. Run golden dataset extraction (75 documents)
3. Compare against stored baseline results
4. Compute regression metrics
5. If any gate fails: alert + hold production extraction
6. If all gates pass: update baseline, continue

### 11.5 Prompt Regression Testing

When any extraction prompt is modified:

1. Run modified prompt against golden dataset
2. Run A/B comparison: old prompt vs new prompt vs golden truth
3. Compute per-field improvement/degradation
4. Log to MLflow experiment with prompt version as parameter
5. Gate: net improvement AND no critical field degradation

---

## 12. Human Review Framework

### 12.1 Review Queue Design

| Queue | Trigger | Reviewer | SLA |
|-------|---------|----------|-----|
| Rate exception | rate_numeric > $50K OR rate change > 50% | Contract analyst | 24h |
| Settlement review | Any settlement extraction | Senior analyst | 48h |
| Hallucination alert | Source check = RED | Any analyst | 24h |
| New provider | First extraction for provider_id | Contract analyst | 72h |
| Low confidence | v6_coverage_pct < 25% | Any analyst | 1 week |
| Audit sample | Random 5% of new extractions | Senior analyst | 1 week |

### 12.2 Inter-Annotator Agreement (IAA)

| Metric | Definition | Target |
|--------|-----------|--------|
| Cohen's kappa (rate correctness) | Agreement on rate-service attribution | >0.85 |
| Cohen's kappa (clause type) | Agreement on clause classification | >0.80 |
| ICC (rate_numeric) | Intra-class correlation for transcribed rates | >0.95 |
| Exact match (effective_date) | % identical date annotations | >0.90 |
| BLEU (clause text) | Textual agreement on clause boundaries | >0.85 |

### 12.3 Human-Review Agreement Scoring

```
human_agreement_score = 
  (system_extractions_approved_by_human) / 
  (total_system_extractions_reviewed)

Per-field agreement:
  rate_numeric_agreement: system_rate == human_rate (within $1)
  service_attribution_agreement: system_service == human_service
  date_agreement: system_date == human_date
  clause_type_agreement: system_type == human_type
```

### 12.4 Calibration Loop

1. Track: for each confidence tier, what % of human reviews approve?
2. If HIGH confidence items rejected >5%: recalibrate threshold
3. If LOW confidence items approved >80%: lower review trigger
4. Target: system confidence correlates with human approval at r > 0.8

---

## 13. Continuous Production Monitoring Framework

### 13.1 Data Quality SLA Metrics

| Metric | SLA | Check Frequency | Alert Threshold |
|--------|-----|-----------------|----------------|
| rate_numeric population rate | >97% | Hourly (post-run) | <95% |
| effective_date_parsed population | >97% | Hourly (post-run) | <95% |
| Source filename coverage | 100% | Every run | <100% |
| NULL explosion (any column) | <5% increase | Every run | >5% delta |
| Row count stability | +/-10% from last run | Every run | >10% change |
| Duplicate rate detection | <5% | Daily | >5% |
| Invalid rate count | <1% | Every run | >1% |
| Provider resolution coverage | 100% | Every run | <100% |

### 13.2 Drift Detection

| Drift Type | Detection Method | Alert Level |
|-----------|-----------------|-------------|
| Distribution drift in rate_numeric | KS test vs historical distribution | WARN if p < 0.05 |
| Category distribution shift | Chi-square test on rate_category proportions | WARN if p < 0.01 |
| NULL rate spike | % NULL increase vs rolling 5-run average | ALERT if > 2 sigma |
| New provider appearing | provider_id not in dim_provider_canonical | INFO |
| Extraction time anomaly | Processing time > 2x rolling average | WARN |
| Token usage anomaly | Average tokens > 2x rolling average | WARN |
| Coverage score drift | Mean v6_coverage_pct drops >10% from baseline | ALERT |

### 13.3 Freshness Monitoring

| Check | Definition | Alert |
|-------|-----------|-------|
| Pipeline last run | Time since last successful pipeline completion | >7 days |
| New PDFs unprocessed | Files in Volume not in file_registry | >0 for >24h |
| Table staleness | Last table write timestamp vs now | >7 days |
| Amendment currency | Newest amendment effective_date vs today | WARN if > 1 year gap |

### 13.4 Production Monitoring Table

```sql
eval_production_monitoring (
  check_timestamp TIMESTAMP,
  check_type STRING,  -- 'data_quality' | 'drift' | 'freshness' | 'anomaly'
  metric_name STRING,
  metric_value DOUBLE,
  threshold DOUBLE,
  status STRING,  -- 'pass' | 'warn' | 'alert' | 'critical'
  details STRING,
  run_id STRING
)
```

---

## 14. Enterprise Trust Scoring Framework

### 14.1 Document-Level Trust Score

```
document_trust_score = 
  0.20 x ocr_quality_score +
  0.25 x extraction_coverage_score +
  0.25 x source_verification_score +
  0.15 x consistency_score +
  0.15 x review_status_score

Where:
  ocr_quality_score = text_length_plausibility x char_density_score
  extraction_coverage_score = v6_coverage_pct / 100
  source_verification_score = (rates_verified + dates_verified) / (rates_total + dates_total)
  consistency_score = 1.0 - (conflicts_found / fields_extracted)
  review_status_score = 
    1.0 if human_approved,
    0.7 if auto_verified,
    0.4 if unverified,
    0.0 if flagged_for_review
```

### 14.2 Field-Level Trust Score

```
field_trust_score =
  source_match x 0.40 +
  attribution_confidence x 0.25 +
  plausibility x 0.15 +
  consistency x 0.10 +
  review_status x 0.10

Trust tiers:
  HIGH (>0.85): Safe for automated use
  MEDIUM (0.60-0.85): Safe with caveats
  LOW (0.35-0.60): Requires human review
  CRITICAL (<0.35): Do not use without verification
```

### 14.3 Platform-Level Trust Score

```
platform_trust_score =
  0.15 x data_quality_sla_compliance +
  0.15 x extraction_precision +
  0.15 x extraction_recall +
  0.10 x hallucination_inverse_rate +
  0.10 x human_review_agreement +
  0.10 x freshness_score +
  0.10 x reproducibility_score +
  0.10 x governance_compliance_score +
  0.05 x monitoring_coverage_score
```

### 14.4 Trust Score Tables

```sql
eval_trust_scores_document (
  source_filename STRING,
  provider_id STRING,
  document_trust_score DOUBLE,
  ocr_quality_component DOUBLE,
  extraction_coverage_component DOUBLE,
  source_verification_component DOUBLE,
  consistency_component DOUBLE,
  review_status_component DOUBLE,
  trust_tier STRING,
  computed_at TIMESTAMP,
  eval_version STRING
)

eval_trust_scores_platform (
  eval_date DATE,
  platform_trust_score DOUBLE,
  data_quality_component DOUBLE,
  precision_component DOUBLE,
  recall_component DOUBLE,
  hallucination_component DOUBLE,
  human_agreement_component DOUBLE,
  freshness_component DOUBLE,
  reproducibility_component DOUBLE,
  governance_component DOUBLE,
  monitoring_component DOUBLE,
  computed_at TIMESTAMP
)
```

---

## 15. Recommended Delta Tables & Metadata Models

### 15.1 Complete Evaluation Table Inventory

| Table | Purpose | Write Pattern |
|-------|---------|---------------|
| `eval_golden_documents` | Document-level ground truth | Manual + version |
| `eval_golden_rates` | Rate-level ground truth | Manual + version |
| `eval_golden_clauses` | Clause-level ground truth | Manual + version |
| `eval_golden_amendments` | Amendment lineage ground truth | Manual + version |
| `eval_golden_queries` | Retrieval/QA benchmark queries | Manual + version |
| `eval_extraction_results` | Per-run extraction evaluation results | Append per run |
| `eval_retrieval_results` | Per-run retrieval evaluation results | Append per run |
| `eval_hallucination_checks` | Per-field hallucination verification | Append per run |
| `eval_regression_results` | Regression test outcomes | Append per change |
| `eval_human_review_decisions` | Human reviewer decisions | Append per review |
| `eval_trust_scores_document` | Document-level trust scores | Overwrite per run |
| `eval_trust_scores_field` | Field-level trust scores (sampled) | Append per run |
| `eval_trust_scores_platform` | Platform-level trust score | Append daily |
| `eval_production_monitoring` | Continuous monitoring events | Append continuous |
| `eval_model_comparison` | A/B model test results | Append per test |
| `eval_prompt_comparison` | A/B prompt test results | Append per test |
| `eval_drift_detection` | Statistical drift test results | Append daily |
| `eval_iaa_scores` | Inter-annotator agreement | Append per session |

### 15.2 Recommended Schema

All evaluation tables should live in a dedicated schema separated from production data:

```sql
CREATE SCHEMA IF NOT EXISTS dev_adb.eval
COMMENT 'Evaluation, benchmarking, and QA framework tables for Contract Intelligence Platform';
```

---

## 16. Recommended Databricks Services

| Need | Databricks Service | Purpose |
|------|-------------------|--------|
| Evaluation orchestration | Lakeflow Jobs | Scheduled eval pipelines |
| Experiment tracking | MLflow | Model/prompt comparison, metrics logging |
| Golden dataset storage | Unity Catalog Delta tables | Versioned, governed ground truth |
| Human review UI | Databricks Apps | Review queue web application |
| Monitoring dashboards | AI/BI Dashboards | Trust scores, drift, SLA compliance |
| Alert notifications | Jobs + webhooks | Threshold-based alerting |
| Evaluation compute | Serverless SQL | Fast evaluation queries |
| LLM-as-judge | Model Serving endpoints | Independent verification model |
| Vector search eval | Databricks Vector Search | Retrieval quality measurement |

---

## 17. Recommended MLflow Integration

### 17.1 Experiment Structure

```
MLflow Experiments:
+-- contract-extraction-eval/
|   +-- run: prompt_v6_claude_sonnet_4.5_2026-05-19
|   |   +-- params: {model, prompt_version, temperature, max_tokens}
|   |   +-- metrics: {precision, recall, f1, halluc_rate, coverage}
|   |   +-- artifacts: {confusion_matrix, per_provider_scores, regression_report}
|   +-- run: prompt_v7_claude_sonnet_4.5_2026-06-01
+-- contract-retrieval-eval/
|   +-- run: vector_search_bge_large_2026-06
|   |   +-- params: {embedding_model, index_type, chunk_strategy}
|   |   +-- metrics: {precision_at_5, recall_at_10, mrr, ndcg}
+-- contract-trust-eval/
|   +-- run: trust_score_weekly_2026-05-19
|       +-- metrics: {platform_trust, extraction_trust, retrieval_trust}
+-- contract-regression/
    +-- run: model_update_check_2026-05-20
        +-- params: {old_model_version, new_model_version}
        +-- metrics: {degraded_fields, improved_fields, gate_status}
```

### 17.2 Key MLflow Metrics

| Category | Metrics |
|----------|--------|
| Extraction quality | rate_precision, rate_recall, rate_f1, date_precision, clause_completeness |
| Hallucination | halluc_rate, source_verification_rate, attribution_accuracy |
| Coverage | v6_coverage_mean, rate_numeric_coverage, date_coverage |
| Trust | document_trust_mean, platform_trust_score |
| Regression | degraded_field_count, improved_field_count, gate_pass_status |
| Retrieval | precision_at_5, recall_at_10, mrr, ndcg_at_10 |

---

## 18. Recommended Evaluation Pipelines

### Pipeline 1: Post-Extraction Evaluation (Every Run)

```
Trigger: After any pipeline extraction run completes
Steps:
  1. Compare extracted rates against source text (hallucination check)
  2. Compute v6_coverage_pct distribution
  3. Check NULL rates for anomalies
  4. Check rate distribution drift
  5. Run plausibility checks
  6. Compute per-document trust scores
  7. Route high-risk items to review queue
  8. Write results to eval_extraction_results
  9. Alert if any SLA metric breached
Duration: ~10 minutes for full corpus
```

### Pipeline 2: Golden Dataset Benchmark (Weekly + On-Change)

```
Trigger: Weekly schedule + any prompt/model change
Steps:
  1. Re-extract golden dataset documents with current config
  2. Compare field-by-field against golden labels
  3. Compute precision, recall, F1 for each field type
  4. Compute clause completeness scores
  5. Compute table reconstruction scores
  6. Compute amendment lineage accuracy
  7. Log to MLflow experiment
  8. Check regression gates
  9. Generate comparison report
  10. Alert if any gate fails
Duration: ~30 minutes (75 documents x 1-2 min each)
```

### Pipeline 3: Production Monitoring (Continuous)

```
Trigger: Hourly schedule
Steps:
  1. Check table row counts vs baseline
  2. Check NULL distributions
  3. Check rate value distributions (KS test)
  4. Check data freshness
  5. Check pipeline last-run timestamp
  6. Check Volume for unprocessed files
  7. Compute platform trust score
  8. Write monitoring events
  9. Alert on threshold breaches
Duration: ~2 minutes
```

### Pipeline 4: Human Review Calibration (Monthly)

```
Trigger: Monthly schedule
Steps:
  1. Sample 50 extractions across confidence tiers
  2. Route to human review queue
  3. Collect review decisions
  4. Compute IAA for dual-reviewed items
  5. Compute system-vs-human agreement
  6. Calibrate confidence thresholds
  7. Update trust score weights
  8. Generate calibration report
Duration: 1-2 weeks (human review dependent)
```

---

## 19. Recommended Governance Controls

### 19.1 Evaluation Governance

| Control | Implementation |
|---------|----------------|
| Golden dataset change approval | PR-style review for any golden label change |
| Evaluation code versioning | All eval code in version-controlled notebooks |
| Metric threshold changes | Documented with justification, requires approval |
| Regression gate overrides | Must be logged with reason and approver |
| Human review calibration | IAA threshold (kappa > 0.80) required before calibration |
| Trust score model changes | Requires validation against historical data |

### 19.2 Audit Trail

Every evaluation run records:

* evaluation_run_id
* triggered_by (schedule, code change, manual)
* code_version (git hash or notebook revision)
* golden_dataset_version
* model_endpoint_version
* prompt_version
* results_summary
* gate_outcomes
* reviewer (if human)

---

## 20. Production Readiness Evaluation Checklist

| # | Requirement | Status | How to Verify |
|---|------------|--------|---------------|
| 1 | Golden dataset exists (>=50 documents annotated) | NOT MET | Count eval_golden_documents |
| 2 | Extraction precision >= 90% on golden set | NOT MET | Run benchmark pipeline |
| 3 | Extraction recall >= 85% on golden set | NOT MET | Run benchmark pipeline |
| 4 | Hallucination rate < 5% on golden set | NOT MET | Run hallucination pipeline |
| 5 | Rate attribution accuracy >= 85% | NOT MET | Run table eval pipeline |
| 6 | Clause completeness >= 90% | NOT MET | Run clause eval pipeline |
| 7 | Human review agreement >= 85% | NOT MET | Complete calibration cycle |
| 8 | Production monitoring operational | NOT MET | Verify monitoring job running |
| 9 | Regression test suite passing | NOT MET | Run regression pipeline |
| 10 | Platform trust score >= 0.75 | NOT MET | Compute trust score |
| 11 | PII masking applied | NOT MET | Verify column masks |
| 12 | Retention policy set (>=365 days) | NOT MET | Check table properties |
| 13 | Run audit log table exists and populated | NOT MET | Check ctl_pipeline_runs |
| 14 | Data quality SLA defined and monitored | NOT MET | Verify SLA config |
| 15 | Freshness check operational | NOT MET | Verify freshness job |

---

## 21. Priority-Wise Implementation Roadmap

### Phase A: Foundation (Weeks 1-4)

| Task | Effort | Impact |
|------|--------|--------|
| Create `dev_adb.eval` schema | 1 day | Enables all eval tables |
| Create eval_golden_documents table | 1 day | Foundation for benchmarks |
| Annotate first 20 golden documents (rates + dates) | 2 weeks | Enables precision/recall |
| Implement automated hallucination detection (source check) | 3 days | Catches fabricated values |
| Implement post-run data quality checks | 2 days | Catches NULL explosions |
| Add model_version + timestamp to extraction | 1 day | Enables version tracking |
| Create MLflow experiment for extraction eval | 1 day | Enables metric tracking |

### Phase B: Core Evaluation (Weeks 5-8)

| Task | Effort | Impact |
|------|--------|--------|
| Complete golden dataset to 50 documents | 2 weeks | Robust benchmarking |
| Implement extraction precision/recall pipeline | 3 days | Quantifies accuracy |
| Implement rate attribution evaluation | 3 days | Catches misattribution |
| Implement clause completeness evaluation | 2 days | Catches truncation |
| Implement regression test suite | 3 days | Catches degradation |
| Create production monitoring job | 2 days | Continuous oversight |
| Implement document-level trust scoring | 2 days | Trust quantification |

### Phase C: Advanced Evaluation (Weeks 9-16)

| Task | Effort | Impact |
|------|--------|--------|
| Implement LLM-as-judge hallucination verification | 1 week | Independent verification |
| Build human review queue + workflow | 2 weeks | Human oversight |
| Implement IAA scoring | 3 days | Calibration capability |
| Build evaluation dashboard | 1 week | Visibility |
| Implement drift detection pipeline | 3 days | Proactive monitoring |
| Implement retrieval evaluation (when vector search exists) | 1 week | Retrieval quality |
| Implement platform trust score | 3 days | Executive metric |

### Phase D: Enterprise Hardening (Weeks 17-24)

| Task | Effort | Impact |
|------|--------|--------|
| Complete golden dataset to 75 documents | 2 weeks | Full coverage |
| Implement regression gates in CI/CD | 1 week | Automated quality gates |
| Build calibration loop | 1 week | Self-improving thresholds |
| Implement RAG/QA evaluation suite | 2 weeks | Answer quality |
| Build comprehensive evaluation report generator | 1 week | Stakeholder reporting |
| Governance audit of evaluation framework | 1 week | Compliance |

---

## 22. Final Enterprise Evaluation Strategy Verdict

The Contract Intelligence Platform requires a **layered, contract-specific evaluation framework** that measures extraction quality at the field level, detects hallucinations through source verification and attribution checking, catches regressions before they reach production, and computes enterprise trust scores that are auditable and actionable.

**The single most impactful evaluation investment is the golden dataset.** Without human-annotated ground truth, no metric can distinguish a confident wrong answer from a correct one. The existing v6_coverage_pct and rate verification are necessary but insufficient — they prove existence, not correctness.

**The three highest-priority evaluation gaps to close:**

1. **Rate-service attribution evaluation** — the largest undetected error class (estimated 15% misattribution) with direct financial impact
2. **Automated hallucination detection via source verification** — can be built immediately without golden data by strengthening the existing string-match approach with context-aware attribution checking
3. **Regression detection pipeline** — essential given that the model endpoint can be updated without notice, silently changing extraction behavior

**Architecture verdict:** The evaluation framework should be built as a **parallel plane** alongside the extraction pipeline — not embedded within it. Evaluation tables, jobs, and dashboards should be independently schedulable, independently governable, and should never block production extraction. However, regression gates should block deployment of prompt/model changes.

The framework scales from the current 3,509 documents to the full 10,372+ corpus, supports multiple model vendors, and evolves as retrieval and QA capabilities are added.

---

## Appendix A: Metric Definitions Reference

| Metric | Formula | Domain |
|--------|---------|--------|
| Precision | TP / (TP + FP) | Extraction |
| Recall | TP / (TP + FN) | Extraction |
| F1 | 2 x (P x R) / (P + R) | Extraction |
| Clause completeness | token_overlap(extracted, golden) / len(golden) | Clause |
| Table reconstruction | 0.3*row_recall + 0.3*cell_acc + 0.2*col_acc + 0.1*header + 0.1*merge | Table |
| Cell accuracy | correct_cells / total_cells | Table |
| Citation precision | correct_citations / total_citations | RAG |
| Precision@k | relevant_in_top_k / k | Retrieval |
| Recall@k | relevant_in_top_k / total_relevant | Retrieval |
| MRR | mean(1/rank_of_first_relevant) | Retrieval |
| NDCG@k | DCG@k / IDCG@k | Retrieval |
| Hallucination rate | ungrounded_claims / total_claims | Trust |
| Grounding score | grounded_claims / total_claims | RAG |
| Source verification rate | source_matched_values / total_values | Hallucination |
| Attribution accuracy | correctly_attributed / total_extracted | Table/Rate |
| Amendment accuracy | correct_supersession_links / total_links | Lineage |
| Current-state accuracy | correct_current_flags / total_flags | Version |
| Provider resolution accuracy | correct_canonical_mapping / total_mappings | Identity |
| Human agreement | system_approved_by_human / total_reviewed | Trust |
| Cohen's kappa | (p_o - p_e) / (1 - p_e) | IAA |
| Platform trust score | weighted composite (see section 14) | Enterprise |
| Trustworthiness score | field_trust weighted by field importance | Enterprise |

---

## END OF PHASE 6 EVALUATION FRAMEWORK
