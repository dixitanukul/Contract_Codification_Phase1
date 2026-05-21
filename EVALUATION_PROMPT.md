# Provider Contract Extraction Pipeline — Deep Evaluation Prompt

**Use this prompt in Databricks Genie Code (notebook assistant) to perform a comprehensive, multi-dimensional validation of the entire contract extraction pipeline: from raw PDF ingestion through LLM extraction to the final 25-table Delta model.**

---

## PROMPT (Copy everything below this line)

---

You are evaluating a production data pipeline that converts 10,372 provider contract PDFs (27 GB, 597 providers, Blue Shield of California) into a 25-table Delta model in `dev_adb.raw`. The pipeline uses `databricks-claude-sonnet-4-5` for semantic extraction.

Perform an **exhaustive 10-phase evaluation** covering every stage from source PDF to final serving tables. For each phase, write and execute SQL/Python validation queries, report findings quantitatively, and flag any issues with severity ratings (CRITICAL / WARNING / INFO).

---

## PHASE 1: SOURCE DATA INTEGRITY

**Objective**: Verify that source PDFs are correctly discovered, accessible, and completely represented in downstream artifacts.

### Checks to perform:
1. **File registry completeness**: Count PDFs in the Volume path `/Volumes/prod_adb/default/ext-data-volume-stmlz/Health_Plan_Ops_Transformation/Provider_Contracts` vs. the cached file registry. Flag any discrepancy.
2. **Provider folder structure**: Count distinct provider folders. Verify folder names parse to valid provider IDs.
3. **File naming conventions**: What percentage of filenames match the expected pattern `providerID_providerName_contractID_docType_version.pdf`? List the top 10 non-conforming filenames.
4. **Document type distribution**: From filenames, tabulate doc_type variants. Confirm presence of Base Agreements, Amendments (1st through Nth), Cover Memos, and Settlements.
5. **File size anomalies**: Find files < 10KB (likely corrupt/empty) and > 100MB (likely image-heavy scans). How many per category?
6. **Extraction coverage**: Compare file_registry count to `json_extract/*.json` count. What percentage of total PDFs have been processed? Which provider folders have 0% extraction?

---

## PHASE 2: OCR & TEXT EXTRACTION QUALITY (Step 1)

**Objective**: Evaluate the 2-tier OCR pipeline (pypdf → ai_parse_document) for completeness and quality of text extraction.

### Checks to perform:
1. **Tier distribution**: What percentage of files were handled by pypdf (Tier 1) vs. ai_parse_document (Tier 2)? What triggers Tier 2 fallback?
2. **Text yield**: Distribution of `text_length` across all parsed files. What's the median, P5, P95? How many files have < 500 characters (MIN_GOOD_CHARS threshold)?
3. **Characters per page ratio**: Compute `text_length / num_pages`. Flag files with < 100 chars/page (MIN_CHARS_PER_PAGE) — these likely have OCR failures or are image-only.
4. **Parse errors**: What error types exist in `parse_error`? How many files have errors? Are encrypted PDFs handled?
5. **Timeout analysis**: How many files hit the 30-second pypdf timeout? Are they all routed to Tier 2 successfully?
6. **Zero-text files**: Any files with `text_length = 0` after both tiers? These represent total extraction failures.
7. **Sample quality check**: Pick 5 random files. Compare extracted text length to PDF page count. Does `chars_per_page` make sense for contract documents (typically 2,000-4,000 chars/page)?

---

## PHASE 3: METADATA PARSING ACCURACY (Step 2)

**Objective**: Validate that filename metadata (provider_id, contract_id, document_type, version) is correctly parsed.

### Checks to perform:
1. **Parse success rate**: What percentage of files have non-NULL values for all 5 metadata fields (provider_id, provider_name, contract_id, document_type, version)?
2. **Provider ID validity**: Are all provider_ids numeric? Any IDs that look like they contain name fragments (misparse)?
3. **Document type classification accuracy**: 
   - How many unique document_type values exist?
   - Map each to the 4 LLM prompt categories (Base, Amendment, Cover Memo, Settlement). What percentage can't be classified?
   - Check for doc_types that SHOULD be Amendments but lack ordinal indicators (e.g., `Amend` without `First`/`Second`/`3rd`).
4. **Version field**: Distribution of version values (v1, v2, v3...). Any files missing versions? Any with version > v5 (unusual)?
5. **Contract ID distribution**: How many unique contract_ids per provider? Any providers with > 20 contracts (possible misparse)?
6. **Cross-reference**: Compare provider_id from filename vs. provider_id from the folder structure. Flag any mismatches.

---

## PHASE 4: LLM EXTRACTION EVALUATION (Step 3)

**Objective**: Assess the quality, completeness, and correctness of Claude Sonnet 4.5's semantic extraction across all document types.

### Checks to perform:
1. **Extraction success rate**: How many JSONs have `extraction_error` field populated? What are the error types (timeout, truncation, malformed JSON, rate limit)?
2. **JSON structural validity**: Read 50 random JSON files. Verify they all have the expected top-level keys: `file_metadata`, `extracted_content`, `validation`.
3. **Document-type-specific completeness**:
   - **Base Agreements** (`doc_role = 'base_agreement'`):
     - Must have: `contract_overview`, `inpatient_rates`, `outpatient_rates`, `parties`, `covered_services`
     - Should have: `key_definitions`, `termination_provisions`, `stop_loss_reinsurance`
     - What % have each section populated?
   - **Amendments** (`doc_role = 'amendment'`):
     - Must have: `amendments_impact`, `effective_date`, rates (if rate amendment)
     - What % have `amendments_impact.supersedes` populated?
   - **Cover Memos** (`doc_role = 'cover_memo'`):
     - Should have: `exhibits_replaced`, `exhibits_added`, `summary_of_changes`
     - What % are stubs vs. substantive?
   - **Settlements** (`doc_role = 'settlement'`):
     - Must have: `settlement_amount`, `settlement_date`, `parties_involved`
     - What % have all 3?
4. **V6 Delta-readiness scores**:
   - Change A (rate_numeric): What % of rate rows have `rate_numeric` populated? Break down by rate_category.
   - Change B (full_clause_text): What % of clauses have verbatim text > 100 chars?
   - Change C (amendments_impact): What % of amendments have supersession tracking?
5. **Hallucination detection**:
   - **Dates**: How many extracted dates fall outside the plausible range (before 1990 or after 2027)? Sample 10 and verify against the raw PDF text.
   - **Rates**: How many rate_numeric values are implausible? (< $0, > $200,000, or exactly round millions)
   - **Provider names**: Do extracted `provider_name` values match the filename-parsed names? Flag divergence > 50% character difference.
   - **Medical codes**: Verify 20 random CPT/HCPCS codes against known valid code ranges.
6. **Chunking effectiveness**: For files > 150K chars that were chunked:
   - How many chunks per file (distribution)?
   - Did any chunks produce duplicate rates when merged?
   - Are section boundaries clean (no mid-sentence splits in extracted content)?
7. **Retry and re-extraction stats**: How many files needed Step 3a-retry? How many triggered Step 3b auto re-extract? What was the recovery rate?

---

## PHASE 5: VALIDATION & ENRICHMENT QUALITY (Steps 4-5)

**Objective**: Verify that post-extraction validation correctly identifies quality issues and that enrichment (medical codes, date verification) is accurate.

### Checks to perform:
1. **Medical code extraction**: Validate regex patterns:
   - CPT codes: 5-digit numeric (10000-99999). What % of extracted codes fall in valid ranges?
   - HCPCS: Alpha + 4 digits. What % match the pattern?
   - Revenue codes: 4-digit (0001-0999). Any false positives (e.g., phone numbers, zip codes)?
2. **Date verification flags**: 
   - How many dates flagged as `unverified`? Can you find the date string in the raw OCR text for 10 random flagged dates?
   - What's the false-positive rate (dates correctly extracted but flagged)?
3. **Rate verification**: 
   - `rate_numeric` vs. rate_text consistency. For 20 random rates, does the numeric match what's written in rate_text?
   - OCR-aware matching: Can you find the rate value in the raw text within ±5% tolerance?
4. **V6 coverage scores**: Distribution of v6_coverage_pct. How does scoring differ by document type? Are cover memos/LOAs unfairly penalized?
5. **Exit code distribution**: From Step 5 quality report — how many files exit 0 (clean), 1 (partial), 2 (critical)?
6. **Content bucket classification**: Tabulate content_bucket values. Is `stub` detection working (routing sheets ≠ failed extractions)?

---

## PHASE 6: CONSOLIDATION LOGIC (Step 6)

**Objective**: Verify that per-file JSONs are correctly merged into per-provider consolidated records with proper amendment chain ordering and rate supersession.

### Checks to perform:
1. **Provider count**: json_consolidated/ file count vs. distinct provider_ids in source files. Should be 1 JSON per provider.
2. **Version de-duplication**: Were any files correctly de-duplicated? (same provider/contract/doctype/effective_date, different versions). Verify highest version was kept.
3. **Amendment chain ordering**: For 5 random providers with > 5 documents:
   - Is the amendment_order monotonically increasing with effective_date?
   - Are there gaps in the chain?
   - Do Base Agreements always have order = 0?
4. **Rate supersession logic**: For each provider, verify:
   - The latest amendment's rates are marked `current`
   - Earlier versions of the same rate (same category + service) are `superseded`
   - No rate appears as both `current` AND `superseded`
5. **Document timeline completeness**: For providers with known Base + multiple Amendments, verify the timeline captures every document in order.
6. **Cross-provider leakage**: Verify no rates from Provider A appear in Provider B's consolidated JSON (check provider_id consistency within each file).

---

## PHASE 7: BASE TABLE INTEGRITY (Steps 11a–11c)

**Objective**: Validate the 8 base tables + dim_provider_canonical for completeness, consistency, and correctness.

### Checks to perform:

#### 7.1 Row Counts & Completeness
```sql
-- Run this exact query and report all results:
SELECT 'tbl_contract_rates_all' AS tbl, COUNT(*) AS rows, COUNT(DISTINCT provider_id) AS providers FROM dev_adb.raw.tbl_contract_rates_all
UNION ALL SELECT 'tbl_contract_documents_master', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.tbl_contract_documents_master
UNION ALL SELECT 'tbl_contract_financial_protections', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.tbl_contract_financial_protections
UNION ALL SELECT 'tbl_contract_regional_factors', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.tbl_contract_regional_factors
UNION ALL SELECT 'tbl_contract_clauses_terms', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.tbl_contract_clauses_terms
UNION ALL SELECT 'tbl_contract_codes_events', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.tbl_contract_codes_events
UNION ALL SELECT 'tbl_contract_parties', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.tbl_contract_parties
UNION ALL SELECT 'tbl_contract_services_quality', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.tbl_contract_services_quality
UNION ALL SELECT 'dim_provider_canonical', COUNT(*), COUNT(DISTINCT provider_id) FROM dev_adb.raw.dim_provider_canonical
```

#### 7.2 Referential Integrity
- Every `provider_id` in base tables must exist in `dim_provider_canonical`. Count orphans per table.
- Every `source_filename` in rates/clauses/parties must exist in `documents_master`. Count orphans.
- Every `canonical_provider_id` (where populated) must match a `primary_provider_id` in dim_provider_canonical.

#### 7.3 NULL Analysis on Critical Columns
For each table, report NULL% on these mandatory columns:
- rates_all: provider_id, source_filename, rate_category, status
- documents_master: provider_id, source_filename, document_type, doc_role
- All tables: provider_id, source_filename

#### 7.4 Duplicate Detection
- Check for exact duplicate rows (all columns identical) in each base table.
- Check for logical duplicates in rates (same provider + source_file + rate_category + service_category + rate_text).

#### 7.5 Identity Resolution Validation
- dim_provider_canonical: 456 IDs → 278 facilities. Verify math: SUM(ids_in_facility * is_primary) should = COUNT(DISTINCT provider_id).
- Spot-check 5 multi-ID facilities: are the merged names truly the same hospital?
- Any orphan provider_ids (in base tables but NOT in dim_provider_canonical)?

---

## PHASE 8: POST-BUILD ENRICHMENT VERIFICATION (Steps 11a-fix, 11c-enrich)

**Objective**: Verify all 13 DQ enrichment steps and the rate supersession fix applied correctly.

### Checks to perform:

#### 8.1 Rate Supersession (Step 11a-fix)
```sql
-- Status distribution should show ~75% superseded, ~25% current
SELECT status, COUNT(*) AS cnt, ROUND(COUNT(*)*100.0/SUM(COUNT(*)) OVER(), 1) AS pct
FROM dev_adb.raw.tbl_contract_rates_all
GROUP BY status
```
- Verify: 0 NULL statuses remain.
- Verify: 0 providers with ALL rates superseded (zero-current safety net).
- Verify: program_normalized has exactly 8 distinct values.

#### 8.2 Date Parsing (Enrichment Step 1)
```sql
-- effective_date_parsed should be DATE type and populated > 90%
SELECT 
  COUNT(*) AS total,
  COUNT(effective_date_parsed) AS parsed,
  ROUND(COUNT(effective_date_parsed)*100.0/COUNT(*), 1) AS pct_parsed,
  MIN(effective_date_parsed) AS earliest,
  MAX(effective_date_parsed) AS latest
FROM dev_adb.raw.tbl_contract_documents_master
```
- Flag if earliest < 1990 (hallucination cleanup should have removed these).
- Flag if latest > 2027 (future hallucination).

#### 8.3 Document Category & Amendment Depth (Step 2)
```sql
SELECT doc_category, COUNT(*) AS cnt
FROM dev_adb.raw.tbl_contract_documents_master
GROUP BY doc_category
```
- Expected categories: base_agreement, amendment, cover_memo, settlement, other.
- Verify amendment_depth is populated and correlates with amendment_order.

#### 8.4 v6_coverage_pct Normalization (Step 10)
```sql
SELECT 
  MIN(v6_coverage_pct) AS min_val,
  MAX(v6_coverage_pct) AS max_val,
  AVG(v6_coverage_pct) AS avg_val,
  PERCENTILE(v6_coverage_pct, 0.5) AS median_val
FROM dev_adb.raw.tbl_contract_documents_master
WHERE v6_coverage_pct IS NOT NULL
```
- MUST be in range 0.0–1.0. If max > 1.0, normalization failed.

#### 8.5 has_full_clause_text (Step 11)
```sql
SELECT has_full_clause_text, COUNT(*) 
FROM dev_adb.raw.tbl_contract_documents_master 
GROUP BY has_full_clause_text
```
- Should NOT be 100% FALSE. Expected ~80-95% TRUE (base agreements and amendments with clauses).

#### 8.6 Pre-1990 Hallucination Cleanup (Step 12)
```sql
-- These should be NULL now (cleaned) but raw strings preserved
SELECT COUNT(*) AS pre_1990_dates
FROM dev_adb.raw.tbl_contract_documents_master
WHERE effective_date_parsed < '1990-01-01'
```
- Must return 0. If > 0, cleanup failed.

#### 8.7 Typed Auxiliary Columns (Step 13)
- Verify `exhibits_replaced_count_int`, `exhibits_added_count_int`, `sections_modified_count_int` are INTEGER type.
- Verify `supersedes_effective_date_parsed` is DATE type.
- Report population rates for each.

#### 8.8 Zero-Current Safety Net
```sql
-- MUST return 0 rows — every provider should have at least 1 current rate
SELECT provider_id, COUNT(*) AS total_rates
FROM dev_adb.raw.tbl_contract_rates_all
GROUP BY provider_id
HAVING SUM(CASE WHEN status = 'current' THEN 1 ELSE 0 END) = 0
```

---

## PHASE 9: SERVING LAYER VALIDATION (Steps 11d–11e)

**Objective**: Verify the 4 Genie tables and 12 serving tables are correctly derived, deduplicated, and documented.

### Checks to perform:

#### 9.1 Genie Table Counts & Consistency
```sql
SELECT 'genie_provider_profile' AS tbl, COUNT(*) FROM dev_adb.raw.tbl_genie_provider_profile
UNION ALL SELECT 'genie_rates_current', COUNT(*) FROM dev_adb.raw.tbl_genie_rates_current
UNION ALL SELECT 'genie_contract_terms', COUNT(*) FROM dev_adb.raw.tbl_genie_contract_terms
UNION ALL SELECT 'genie_amendment_timeline', COUNT(*) FROM dev_adb.raw.tbl_genie_amendment_timeline
```
- provider_profile rows should = dim_provider_canonical facilities (WHERE is_primary_id = TRUE).
- amendment_timeline rows should = documents_master rows.

#### 9.2 Deduplication Effectiveness
- Compare `tbl_contract_rates_all WHERE status='current'` count to `tbl_genie_rates_current` count.
- The dedup ratio should be > 1.0x (serving has fewer rows due to dedup).
- Verify ZERO exact duplicates in genie_rates_current (full business key including program + network):
```sql
-- True duplicates: same rate for same provider/category/service/text/date/program/network
SELECT provider_name, rate_category, service_category, rate_text, effective_date_parsed, program_normalized, network, COUNT(*) AS dupes
FROM dev_adb.raw.tbl_genie_rates_current
GROUP BY ALL
HAVING COUNT(*) > 1
```
- NOTE: Rows differing ONLY in program_normalized or network are NOT duplicates — they represent distinct contractual rates for different programs/networks. The dedup PARTITION BY uses `program_normalized` (not raw program) to collapse variant spellings like "HMO Group"/"HMO IFP"/"POS" → "Commercial".

#### 9.3 Column Comments Completeness
```sql
-- Check all 4 Genie tables have 100% column comments
SELECT table_name, 
  COUNT(*) AS total_cols,
  COUNT(comment) AS commented_cols,
  ROUND(COUNT(comment)*100.0/COUNT(*), 1) AS pct
FROM information_schema.columns
WHERE table_schema = 'raw' AND table_catalog = 'dev_adb'
  AND table_name LIKE 'tbl_genie_%'
GROUP BY table_name
```
- Target: 100% commented columns on all Genie tables.

#### 9.4 Serving Layer Completeness
```sql
-- Verify all 12 serving tables + 2 dimensions exist
SHOW TABLES IN dev_adb.raw LIKE 'tbl_serving_*'
```
Expected 12 tables:
- tbl_serving_rate_card
- tbl_serving_stop_loss
- tbl_serving_amendment_history
- tbl_serving_parties
- tbl_serving_regional_factors
- tbl_serving_provider_summary
- tbl_serving_capitation_card
- tbl_serving_case_rate_card
- tbl_serving_outpatient_card
- tbl_serving_services_coverage
- dim_service_category
- dim_network

#### 9.5 Provider Summary Accuracy
For `tbl_serving_provider_summary`:
- Recompute `avg_inpatient_per_diem` from base data (rates WHERE status='current' AND rate_category='inpatient_per_diem' AND rate_numeric BETWEEN 100 AND 50000, joined through dim_provider_canonical). Compare to stored value for 10 random providers. Divergence > $1 is a WARNING (V8.1 fix ensures max $0.01 divergence since both now use the same base query).
- Verify `total_amendments` excludes sentinel values (amendment_order < 100).
- Verify `has_stop_loss` matches existence in tbl_contract_financial_protections.

#### 9.6 Rate Card Validation
For `tbl_serving_capitation_card`, `tbl_serving_case_rate_card`, `tbl_serving_outpatient_card`:
- Verify rate_category filter is correct (capitation only in capitation_card, etc.)
- Verify all rows have `status = 'current'`
- Verify provider_name comes from dim_provider_canonical (not raw provider_name)

#### 9.7 dim_network Mapping
- Verify all 185+ raw network values map to one of the canonical network names.
- Verify no NULL canonical_network values.
- Spot-check: "HMO Assigned" → "Assigned", "PPO Network" → "PPO", etc.

---

## PHASE 10: END-TO-END TRACEABILITY & CROSS-LAYER CONSISTENCY

**Objective**: Verify data flows correctly from PDF → JSON → Base → Genie → Serving with no information loss or corruption.

### Checks to perform:

#### 10.1 Single-Provider Deep Trace
Pick one provider with a rich history (> 10 documents, multiple rate types, amendments). Trace completely:
1. Count their PDFs in file_registry
2. Count their JSON files in json_extract/
3. Count their rows in documents_master
4. Verify their amendment chain in consolidated JSON
5. Count their rates in base (current + superseded)
6. Count their rates in Genie (only current, deduped)
7. Verify their profile in serving layer
8. Confirm all values are consistent across layers

#### 10.2 Rate Value Traceability
Pick 5 random rates from `tbl_genie_rates_current`. For each:
1. Find the source_filename
2. Read the JSON file
3. Find the rate in `extracted_content`
4. Verify `rate_numeric` matches
5. Verify `rate_text` is identical
6. Check if the rate exists in the original PDF text (search raw OCR text)

#### 10.3 Cross-Layer Count Consistency
```sql
-- These ratios should make logical sense
SELECT
  (SELECT COUNT(*) FROM dev_adb.raw.tbl_contract_documents_master) AS base_docs,
  (SELECT COUNT(*) FROM dev_adb.raw.tbl_genie_amendment_timeline) AS genie_timeline,
  (SELECT COUNT(*) FROM dev_adb.raw.tbl_serving_amendment_history) AS serving_history,
  (SELECT COUNT(*) FROM dev_adb.raw.tbl_contract_rates_all WHERE status = 'current') AS base_current_rates,
  (SELECT COUNT(*) FROM dev_adb.raw.tbl_genie_rates_current) AS genie_current_rates,
  (SELECT COUNT(DISTINCT provider_id) FROM dev_adb.raw.dim_provider_canonical) AS all_provider_ids,
  (SELECT COUNT(*) FROM dev_adb.raw.dim_provider_canonical WHERE is_primary_id = TRUE) AS facilities,
  (SELECT COUNT(*) FROM dev_adb.raw.tbl_genie_provider_profile) AS genie_profiles,
  (SELECT COUNT(*) FROM dev_adb.raw.tbl_serving_provider_summary) AS serving_profiles
```
Expected relationships:
- base_docs = genie_timeline = serving_history
- genie_current_rates ≤ base_current_rates (dedup removes some)
- facilities = genie_profiles = serving_profiles

#### 10.4 Program Normalization Consistency
- Verify `program_normalized` in base matches `program_normalized` in Genie tables.
- Verify exactly 8 distinct values: Commercial, CalPERS, Medicare, Medi-Cal, Dual Eligible, Commercial Trio, All Programs, Other.

#### 10.5 Temporal Consistency
- No rate should have `effective_date_parsed` AFTER the document's `effective_date_parsed`.
- Amendment_order in documents should correlate with effective_date chronological order (with some tolerance for misfiled docs).

#### 10.6 Financial Sanity Checks
```sql
-- Rate distribution should make sense for California healthcare
SELECT 
  rate_category,
  COUNT(*) AS cnt,
  ROUND(AVG(rate_numeric), 0) AS avg_rate,
  ROUND(PERCENTILE(rate_numeric, 0.5), 0) AS median_rate,
  MIN(rate_numeric) AS min_rate,
  MAX(rate_numeric) AS max_rate
FROM dev_adb.raw.tbl_genie_rates_current
WHERE rate_numeric IS NOT NULL AND rate_numeric > 0
GROUP BY rate_category
ORDER BY cnt DESC
```
Expected ranges (California 2020-2025):
- Inpatient per diem: $500–$10,000 (median ~$1,500–$3,000)
- Case rates: $5,000–$100,000 (transplants can be higher)
- Capitation PMPM: $5–$500
- Outpatient: $100–$5,000

Flag any category where median is outside these ranges.

---

## EVALUATION REPORT FORMAT

After executing all checks, produce a structured report:

```
═══════════════════════════════════════════════════════════════════
  PIPELINE EVALUATION REPORT
  Generated: [timestamp]
  Scope: [X] tables, [Y] source files processed
═══════════════════════════════════════════════════════════════════

OVERALL SCORE: [X/100]

CRITICAL ISSUES: [count]
  • [issue description + evidence + remediation]

WARNINGS: [count]
  • [issue description + evidence + suggested fix]

INFO: [count]
  • [observation]

─── PHASE SCORES ───
  Phase 1 (Source Integrity):      [X/10]
  Phase 2 (OCR Quality):           [X/10]
  Phase 3 (Metadata Parsing):      [X/10]
  Phase 4 (LLM Extraction):        [X/10]
  Phase 5 (Validation):            [X/10]
  Phase 6 (Consolidation):         [X/10]
  Phase 7 (Base Tables):           [X/10]
  Phase 8 (Enrichment):            [X/10]
  Phase 9 (Serving Layer):         [X/10]
  Phase 10 (Cross-Layer):          [X/10]

─── KEY METRICS ───
  PDF Coverage:          [X/10,372] ([Y]%)
  Extraction Success:    [X]%
  rate_numeric Coverage: [X]%
  Referential Integrity: [X]%
  Duplicate Rate:        [X]%
  Date Parse Rate:       [X]%
  Column Comment Coverage: [X]%
  Zero-Current Providers: [X] (must be 0)
  Tables Present:        [X/25]
  Pre-1990 Hallucinations: [X] (must be 0)

─── RECOMMENDATIONS ───
  Priority 1: [action]
  Priority 2: [action]
  Priority 3: [action]
```

---

## EXECUTION NOTES

1. **Run this evaluation AFTER the pipeline completes** (all steps through 11e).
2. **The schema is `dev_adb.raw`** — all tables live there.
3. **Use `display()` for all query results** so outputs are visible.
4. **Break into multiple cells** — one per phase for clarity.
5. **If any check requires reading JSON files**, they live at:
   - Individual: `/Workspace/Users/adixit01@blueshieldca.com/Contract_Codification_Optimized/json_extract/`
   - Consolidated: `/Workspace/Users/adixit01@blueshieldca.com/Contract_Codification_Optimized/json_consolidated/`
6. **Checkpoint parquets** are at:
   - `/Workspace/Users/adixit01@blueshieldca.com/Contract_Codification_Optimized/checkpoints/step1_parsed_v2.parquet`
   - `/Workspace/Users/adixit01@blueshieldca.com/Contract_Codification_Optimized/checkpoints/step2_with_metadata_v2.parquet`
7. **The pipeline notebook** is at ID `3871536132218218`.
8. **Current state**: 3,518/10,372 PDFs processed (33.9%), 25 tables built.
9. **Do NOT modify any tables** — this is read-only evaluation.
10. **Time budget**: This evaluation should take 10-15 minutes to run all queries.

---

## CONTEXT: KNOWN ISSUES (Already Fixed in V8 + V8.1)

These were previously identified and fixed. Verify they REMAIN fixed:

### V8 Fixes (2026-05-13 morning):
- v6_coverage_pct was stored as 0-100 (now normalized to 0.0-1.0)
- has_full_clause_text was always FALSE (now computed from actual clause content)
- 44 providers had zero current rates (safety net now promotes latest-doc rates)
- 40 documents had pre-1990 hallucinated dates (now nulled, raw preserved)
- 5 serving tables were never created (now all 12 + 2 dims exist)
- Column comments were 69% complete (now 100% on Genie tables)

### V8.1 Fixes (2026-05-13 afternoon):
- **genie_rates_current duplicates**: S2 dedup PARTITION BY used raw `program` (e.g., "HMO Group", "HMO IFP", "POS") instead of `program_normalized` ("Commercial"). Rates with different raw program spellings that normalize to the same bucket survived as separate rows. Fix: changed to `COALESCE(r.program_normalized, 'Commercial')` — eliminated 166 excess rows, zero true duplicates remain.
- **tbl_serving_provider_summary avg divergence**: `avg_inpatient_per_diem` was computed from `tbl_serving_rate_card` (which applies service-line ROW_NUMBER dedup). This caused 47 providers to diverge >$50 from the base table recompute. Fix: rates CTE now computes directly from `tbl_contract_rates_all` WHERE status='current' AND rate_category='inpatient_per_diem' AND rate_numeric BETWEEN 100 AND 50000. Max divergence now $0.01.

### Expected counts after V8.1:
- tbl_genie_rates_current: ~14,637 rows (was 14,803 before dedup fix)
- True duplicate groups (full business key): 0
- Provider summary divergences > $50: 0

---

## BONUS: SPOT-CHECK WITH ORIGINAL PDF

If you want maximum rigor, pick ONE source PDF and:
1. Read its raw OCR text from step1_parsed_v2.parquet
2. Read its extracted JSON from json_extract/
3. Manually verify 3 rates, 2 dates, and 1 clause match between OCR text and extraction
4. Trace those values through to the final serving tables
5. Confirm nothing was lost or transformed incorrectly

This single end-to-end trace is the strongest possible validation of pipeline correctness.
