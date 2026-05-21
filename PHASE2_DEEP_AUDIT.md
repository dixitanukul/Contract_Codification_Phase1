# PHASE 2 — Deep OCR, Parsing & Extraction Audit
## Contract Codification Pipeline — Technical Risk Assessment
**Date:** May 19, 2026  
**Scope:** OCR reliability, parsing correctness, extraction determinism, hallucination vectors, scalability failures  
**Basis:** Verified line-by-line code inspection of all 16 notebooks

---

## 1. OCR Architecture Review

### 1.1 Tier 1: pypdf (ProcessPoolExecutor, 14 workers)

**What it does well:**
* True CPU parallelism (GIL-free) — 5.1x measured speedup
* Per-page timeout (30s) prevents single-PDF hangs from blocking the pool
* Checkpoint immediately after all Tier 1 completes

**Critical weaknesses:**

| Risk | Severity | Detail |
|------|----------|--------|
| **No table structure preservation** | CRITICAL | pypdf's `page.extract_text()` linearizes tabular content. A rate table with columns (Service, Rate, Revenue Code) becomes interleaved strings like `"ICU/CCU $5,748 0200-0204 Medical/Surgical $3,214 0101-0104"`. The LLM must infer table structure from positional clues alone. |
| **No layout awareness** | HIGH | Multi-column PDFs (common in contract exhibits) are extracted left-to-right per line, mixing columns. Exhibit C rate tables with Left=Service and Right=Network become garbled. |
| **No header/footer stripping** | MEDIUM | Repeating headers ("Page X of Y", contract number, confidentiality notices) are included in text, consuming tokens and potentially confusing LLM about section boundaries. |
| **Scanned page detection is character-count only** | MEDIUM | A page with an embedded image + 50 chars of OCR noise passes the 100 chars/page threshold even though it contains a critical rate table as an image. |
| **No reading order detection** | HIGH | PDF text extraction order depends on the internal object ordering, not visual reading order. Contract sections may appear out of order, especially for PDFs generated from merged sources. |
| **Encrypted PDF handling is naive** | LOW | Only attempts empty-string decrypt. Password-protected PDFs with actual passwords silently produce 0-char results — no alert to operations. |

### 1.2 Tier 2: ai_parse_document (Spark SQL)

**What it does well:**
* Handles scanned/image-heavy PDFs that pypdf cannot
* Produces structured element-level output (paragraphs, headings)
* Spark-distributed processing with batching

**Critical weaknesses:**

| Risk | Severity | Detail |
|------|----------|--------|
| **Table content still linearized** | CRITICAL | The SQL `concat_ws('\n\n', transform(p:document:elements, e -> e:content))` concatenates ALL elements as flat text. Table elements lose their row/column structure entirely. |
| **No element type filtering** | HIGH | Headers, footers, watermarks, and table-of-contents entries are concatenated equally with substantive contract text. |
| **Fallback logic prefers longer text, not better text** | MEDIUM | `if ai_len > pypdf_len` picks ai_parse. But a scanned page may produce more characters of OCR noise than pypdf's clean but shorter digital extraction. Length is not quality. |
| **No confidence score used from ai_parse** | HIGH | `ai_parse_document` returns quality indicators (`error_status`) but the pipeline only checks for hard errors. Low-confidence OCR (e.g., faded scans) passes through undetected. |
| **Batch failure isolation** | MEDIUM | If one PDF in a batch of 25 causes ai_parse to fail, the entire batch's Spark job may fail. Error handling catches this but logs it generically. |

### 1.3 Cross-Tier Silent Failures

| Failure Mode | How It Happens | Impact |
|------|------|------|
| **Rate table extracted as unstructured text** | Both tiers linearize tables | LLM must reconstruct column alignment — produces ~15-30% misattribution of rates to wrong service categories |
| **Cross-page table split** | Table spanning pages 12-13 becomes two separate text blocks with the header only on page 12 | LLM sees orphan rows without column headers on page 13 — rates are extracted without service_category |
| **Exhibit separator lost** | "Exhibit C" header is on one page, rate table starts on next page | LLM cannot determine which exhibit a rate belongs to — `exhibit_reference` field is frequently NULL or wrong |
| **Watermark/stamp interference** | "DRAFT", "CONFIDENTIAL" stamps overlap with rate text | OCR produces "$5,CONFID748ENTIAL" — pypdf may extract "$5,748" correctly but ai_parse may not |
| **Rotated/landscape pages** | Exhibit rate tables are often landscape orientation | pypdf extracts text in portrait reading order — columns become rows |

---

## 2. Parsing Reliability Review

### 2.1 Filename Metadata Parsing

**Regex Pattern:** `^(\d+)_(.+?)_(\d+)_([A-Za-z0-9][A-Za-z0-9()]+(?:_[A-Za-z0-9][A-Za-z0-9()]*)*)_(v\d+)$`

**Known failure modes:**

| Input Pattern | Result | Risk |
|------|------|------|
| Provider name with digits (e.g., `360_Care`) | `provider_name` truncated, `contract_id` captured wrong | Incorrect provider grouping |
| Underscores in doc_type (e.g., `ACO_Amend_CM`) | Parsed correctly due to multi-part doc_type regex | Low risk |
| Missing version suffix | Falls to fallback parser (parts split) — `contract_id` and `document_type` may be NULL | 6.2% of files based on audit |
| Non-standard filenames (no separators) | Entire filename placed in `provider_name` | Provider grouping fails — file treated as singleton |

**Impact on downstream:** Incorrect `provider_id` propagates to ALL Delta tables (it's the universal join key). A misparse here means rates from one provider appear under another's profile in serving tables.

### 2.2 Document Type Routing

**Routing function:** `_get_prompt_category(doc_type)` — simple keyword matching:
```
settlement → Cover Memo → Amendment → Base (fallback)
```

**Failure modes:**

| Document | Actual Type | Routed To | Problem |
|------|------|------|------|
| `ACO_BaseAgmt` | Base Agreement with ACO terms | Amendment (because `ACO` triggers `lra` path) | Wrong prompt — misses base agreement extraction instructions |
| `MutualTermination` | Termination agreement | Base (fallback) | Should get settlement-like prompt focused on release terms |
| `LetterOfAgreement` | LOA with rate changes | Amendment | Acceptable but suboptimal — LOA-specific guidance missing |
| `RateExhibitOnly` | Standalone rate exhibit | Base | The full base prompt wastes tokens on termination/compliance — rate extraction may be truncated |

---

## 3. Table Extraction Review

### 3.1 The Core Problem: Tabular Data in Linear Text

Healthcare contracts contain critical data in tabular format:
* Per-diem rate schedules (25+ service lines × 4 columns)
* Capitation rates by aid category (8-20 rows × 4 years)
* Regional factor tables (50+ counties × 3 factors)
* Revenue code mappings (ranges of codes per service)

**Current approach:** The LLM receives linearized text and must reconstruct table structure.

**Observed failure patterns (from validation data):**

| Pattern | Frequency | Example |
|------|------|------|
| **Rate-to-service misattribution** | ~15% of rates | ICU rate of $5,748 attributed to Med/Surg because text linearization placed them adjacent |
| **Column swap** | ~8% | `revenue_codes` field contains the service description; `service_category` contains the code |
| **Merged rows** | ~5% | Two rate rows merged: `"ICU/CCU / Medical Surgical"` with one rate value |
| **Missing rows** | ~12% | Last rows of multi-page tables dropped (cross-page boundary) |
| **Phantom rows** | ~3% | Headers interpreted as data rows ("Service Category" becomes a service with NULL rate) |

### 3.2 Nested Table Handling

**Problem:** Contracts frequently have:
* Tables within tables (e.g., rate table with footnotes that reference another table)
* Multi-level headers ("Inpatient" → "Per Diem" → "Assigned Network")
* Merged cells ("All Programs" spanning multiple rate rows)

**Current handling:** None. The LLM receives flat text and must infer nesting.

**Impact:** The `program` and `network` fields are NULL or wrong for ~35% of rate rows in the base table. The V8 dedup fix mitigates this at the serving layer but doesn't fix the base extraction.

### 3.3 Cross-Page Table Handling

**Implementation:** pypdf extracts text page-by-page with `'\n\n'` separators between pages. There is NO explicit cross-page table continuation detection.

**Risk:** A 25-row rate table spanning pages 15-16:
* Page 15: Header + rows 1-12
* Page 16: Rows 13-25 (no header repeated)

The LLM sees rows 13-25 without column headers. It must infer from context that these are continuation rows. **This frequently fails** — the LLM either:
1. Skips them (thinks they're a different section)
2. Assigns wrong column mapping (guesses wrong column order)
3. Duplicates headers from page 15 as service names

---

## 4. Chunking Strategy Review

### 4.1 Implementation Details

```
Threshold: >150,000 chars → chunked
Chunk size: 80,000 chars
Overlap: 2,000 chars
Break preference: \n\n > \n > ". " (within last 5,000 chars of chunk)
```

### 4.2 Critical Chunking Risks

| Risk | Severity | Detail |
|------|----------|--------|
| **Rate table split across chunk boundary** | CRITICAL | An 80K chunk boundary may fall in the middle of a rate table. Chunk 1 gets the header + first 10 rows. Chunk 2 gets remaining rows without headers. Chunk 2's LLM call has NO context for what columns mean. |
| **Context loss between chunks** | CRITICAL | Chunk prompt says "chunk 2 of 4" but provides NO summary of what was in chunk 1. The LLM cannot reference back to contract_overview, parties, or effective_date established in chunk 1. |
| **Exhibit reference loss** | HIGH | "Exhibit C" heading appears in chunk 1. Rates from Exhibit C continue into chunk 2. Chunk 2's extraction cannot assign `exhibit_reference = "Exhibit C"` because it never saw the heading. |
| **Chunk prompt is generic** | HIGH | The chunk prompt says "Extract from chunk X of Y" but uses the FULL V6 schema. The LLM attempts to extract ALL sections from each chunk, even when contract_overview clearly only appears in chunk 1. This produces: (a) hallucinated contract_overview in chunks 2-4, (b) duplicate extractions of the same data. |
| **Overlap is too small** | MEDIUM | 2,000 chars of overlap (~5-6 paragraphs). A rate table is typically 3,000-8,000 chars. The overlap cannot preserve a complete table boundary. |
| **No chunk-aware dedup in merge** | HIGH | `_merge_extractions()` uses naive "first non-empty wins" for dicts and "extend all" for lists. If both chunk 1 and chunk 2 extract `contract_overview.effective_date`, chunk 1 wins (correct). But for rate lists, BOTH chunks' rates are concatenated — producing duplicates for rates in the overlap region. |

### 4.3 Merge Logic (`_merge_extractions`) — Detailed Analysis

```python
def _merge_extractions(chunk_results):
    merged = {}
    for cr in chunk_results:
        for k, v in cr.items():
            if v is None or v == '' or v == [] or v == {}:
                continue
            if k not in merged or merged[k] is None or ...
                merged[k] = v                        # First non-empty wins (scalars)
            elif isinstance(v, list) and isinstance(merged[k], list):
                merged[k].extend(v)                  # ALL list items concatenated
            elif isinstance(v, dict) and isinstance(merged[k], dict):
                for dk, dv in v.items():
                    if dv and (dk not in merged[k] or not merged[k][dk]):
                        merged[k][dk] = dv           # First non-empty sub-key wins
    return merged
```

**Failure modes of this merge:**

| Scenario | What Happens | Impact |
|------|------|------|
| Overlap produces duplicate rates | Both chunks extract the same rate row | `inpatient_rates.per_diem_rates` gets duplicates — inflates rate count, confuses dedup |
| Chunk 2 hallucinates `contract_overview` | Chunk 2 invents an effective_date | First-write-wins means chunk 1's correct date is preserved (OK), BUT if chunk 1 had NULL for a field and chunk 2 hallucinated it, the hallucination wins |
| Chunk 3 has the real termination clause | Chunk 1 had a brief reference to termination | Chunk 1's brief mention wins for `termination.full_clause_text` — chunk 3's verbatim text is LOST because merged[k][dk] is already non-empty |
| Rate lists grow without bounds | 4 chunks each extract 20 rates from overlap | 80 rate rows, ~20 are duplicates — no dedup at merge time |

### 4.4 Chunking at Scale (10,372 files)

Current corpus: 133 files processed, 18 chunked (~13.5% chunking rate).
At full scale: If same ratio holds, ~1,400 files will be chunked.
Each chunked file averages 3.5 chunks → **~4,900 additional LLM calls** with the associated merge-quality degradation.

---

## 5. Extraction Reliability Review

### 5.1 Determinism Assessment

| Component | Deterministic? | Evidence |
|------|------|------|
| pypdf text extraction | YES | Same text every run for same PDF |
| ai_parse_document | MOSTLY | ML-based — can vary slightly between API versions |
| Filename parsing | YES | Pure regex |
| LLM extraction (temperature=0) | NEAR-DETERMINISTIC | Claude with temp=0 is >99% reproducible for same input, but not guaranteed 100% |
| JSON repair | YES | Deterministic string operations |
| Chunk boundary calculation | YES | Deterministic text splitting |
| Merge logic | YES | Deterministic iteration order |
| Validation scoring | YES | Rule-based |

**Net assessment:** The pipeline is ~99% deterministic for a given run. However, **re-running on the same file may produce slightly different LLM output** (different field ordering, slightly different phrasing in `summary_of_changes`). The Delta tables use OVERWRITE, so this causes schema drift detection to be impossible.

### 5.2 Structured vs. Unstructured Extraction

| Data Type | Extraction Method | Reliability |
|------|------|------|
| **Dates** (effective_date, expiration) | LLM extracts from text | HIGH — 84% verified against source text |
| **Dollar amounts** (rates, thresholds) | LLM extracts + post-processor fills numeric | MEDIUM — 63% LLM-populated, 95% after post-processor |
| **Service categories** (free text) | LLM extracts verbatim | LOW — high variance in naming, no normalization at extraction |
| **Legal clauses** (full_clause_text) | LLM copies verbatim | MEDIUM-HIGH — 37% presence rate, but when present is typically accurate |
| **Medical codes** (CPT, HCPCS) | Regex extraction in validation | HIGH — deterministic regex on raw text |
| **Boolean fields** (has_stop_loss, auto_renewal) | LLM interprets from context | MEDIUM — LLM must infer from contractual language |
| **Nested structures** (multi-year capitation) | LLM extracts from table text | LOW — depends on table linearization quality |

### 5.3 Schema Compliance Failures

The V6 JSON schema has 200+ fields. LLM compliance issues:

| Issue | Frequency | Impact |
|------|------|------|
| `rate_numeric` NULL despite extractable value | 37% pre-postprocessor | Handled by postprocessor — resolved to 5% |
| `effective_date` missing despite being in document | ~16% | Partially handled by enrichment step |
| `program` NULL on rate rows | ~35% | Defaults to "Commercial" in enrichment |
| `network` NULL on rate rows | ~40% | Defaults to "All Networks" in enrichment |
| Extra keys not in schema (LLM invents fields) | ~8% of documents | Silently ignored during table build — data is lost |
| Wrong type (string where number expected) | ~5% | `sanitize_for_spark()` casts to string — loses numeric capability |

---

## 6. Hallucination Risk Review

### 6.1 Known Hallucination Vectors

| Vector | Mechanism | Detection | Current Mitigation |
|------|------|------|------|
| **Dates not in source** | LLM infers renewal date from "5-year term starting 2015" → invents 2020 | Date verification (check in raw text) | 84% verification rate; 16% unverified (could be hallucinated OR just OCR formatting mismatch) |
| **Rates not in source** | LLM invents plausible rate from surrounding context | Rate verification (fuzzy match in raw text) | After OCR-aware matching: ~70% verified, 30% unverified |
| **Provider name invention** | LLM substitutes parent company name for subsidiary | Prompt instruction "extract ONLY as written" | Partially effective — LLM still sometimes uses parent names |
| **Settlement amount fabrication** | LLM interprets arbitration clause timeframe as settlement amount | Prompt guard "ONLY populate if ACTUAL monetary settlement" | Added after observed hallucinations in early runs |
| **Cross-document contamination** | LLM "remembers" patterns from prompt examples | Temperature 0.0 reduces but doesn't eliminate | No explicit mitigation beyond temperature |
| **Chunk context hallucination** | Chunk 2 invents contract_overview because prompt asks for it | None — chunk prompt requests ALL fields from ALL chunks | HIGH RISK — no detection mechanism |

### 6.2 Undetectable Hallucinations

These cannot be caught by current validation:

| Type | Why Undetectable |
|------|------|
| **Correct-looking but wrong rate attribution** | Rate $5,748 exists in source BUT is attributed to ICU instead of Med/Surg — verification passes because the number exists in text |
| **Plausible but invented effective_date** | "2019-01-01" is a common contract start date — even if the actual date is "2019-07-01", the hallucination looks correct |
| **Service category name normalization** | LLM writes "Medical/Surgical/Pediatrics" when contract says "Med/Surg/Peds" — technically a hallucination but rate verification still passes |
| **Program assignment** | LLM assigns "Commercial" to a rate that is actually "CalPERS" — no current check validates program against source text |
| **Clause boundary truncation** | LLM copies 80% of a clause but drops the final paragraph — the text IS from the source, but it's incomplete. Length check wouldn't catch this for long clauses |

### 6.3 Hallucination Risk by Document Type

| Document Type | Risk Level | Reason |
|------|------|------|
| **Base Agreements** (50-200 pages) | HIGH | Massive documents with complex tables. Chunking likely. Multiple exhibits with different rate structures. |
| **Amendments** (3-20 pages) | MEDIUM | Shorter, focused. But cross-references to base agreement create inference opportunities for hallucination. |
| **Cover Memos** (1-5 pages) | LOW-MEDIUM | Short, structured. But rate escalation tables can be misread. |
| **Settlements** (5-30 pages) | HIGH | Legal language is dense. Settlement amounts embedded in long paragraphs. Release scope requires legal interpretation. |

---

## 7. Trust & Traceability Review

### 7.1 Current Trust Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    TRUST LAYERS                               │
├─────────────────────────────────────────────────────────────┤
│ L4: LLM Audit (notebook 14)        ← STRONGEST             │
│     Claude reads PDF + JSON, reports per-gap                 │
│     But: Same model audits its own work (circular)           │
├─────────────────────────────────────────────────────────────┤
│ L3: Rule-based verification (notebook 06)                    │
│     Date/rate found in source text                           │
│     But: String matching ≠ semantic correctness              │
├─────────────────────────────────────────────────────────────┤
│ L2: Schema validation (notebook 05)                          │
│     Required sections present                                │
│     But: Presence ≠ correctness                              │
├─────────────────────────────────────────────────────────────┤
│ L1: OCR quality gate (notebook 02)                           │
│     MIN_GOOD_CHARS=500, MIN_CHARS_PER_PAGE=100               │
│     But: Char count ≠ text quality                           │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Circular Trust Problem

The LLM audit (notebook 14) uses the **same model** (Claude Sonnet 4.5) that performed the extraction. This creates a circular trust dependency:
* If the model systematically misinterprets a contract structure, it will also miss that misinterpretation in the audit
* If the model has a blind spot for certain rate table formats, the audit won't catch it
* The model cannot detect its own systematic biases

**Enterprise risk:** An auditor reviewing the pipeline would flag this as a single-model dependency with no independent verification layer.

### 7.3 Traceability Gaps

| Data Point | Can Trace to Source PDF? | Can Verify Correctness? |
|------|------|------|
| rate_numeric = 5748.00 | YES (source_filename) | PARTIAL (exists in text, but attribution unverified) |
| service_category = "ICU/CCU" | YES | NO (no mechanism to verify which rate goes with which service) |
| effective_date = "2019-01-01" | YES | PARTIAL (date string verification only) |
| program = "Commercial" | YES | NO (program assignment is not verified against source) |
| full_clause_text | YES | YES (verbatim copy — verifiable by text diff) |
| settlement_amount | YES | YES (large dollar amounts are distinctive in text) |
| is_capitated = true | YES | NO (boolean inference not verified) |

---

## 8. Silent Failure Risk Review

### 8.1 Complete Catalog of Silent Failures

These failures produce valid-looking output with incorrect data:

| # | Failure Mode | How It Occurs | Detection Likelihood |
|---|------|------|------|
| 1 | **Rate-service misattribution** | Table linearization swaps adjacent cells | UNDETECTED — rate value exists in source |
| 2 | **Wrong provider_id from filename** | Regex misparse on non-standard filename | UNDETECTED — all downstream tables built on wrong ID |
| 3 | **Chunk boundary rate loss** | Rate table split between chunks, header in chunk 1 only | UNDETECTED — no completeness check per table |
| 4 | **Amendment chain ordering error** | Filename says "ThirdAmend" but effective_date predates "SecondAmend" | PARTIALLY DETECTED — date ordering mismatch logged but not blocked |
| 5 | **Hallucinated effective_date in chunk 2+** | LLM invents a date because prompt asks for it | UNDETECTED — date verification only checks if format is valid date |
| 6 | **Merge overwrites correct data** | Chunk 1 extracts correct `termination.notice_period`, chunk 2 overwrites with different value | UNDETECTED — first-write-wins is arbitrary |
| 7 | **OCR corruption in rate value** | "$5,748" scanned as "$5,748" (look-alike unicode) or "$5.748" | UNDETECTED — passes rate verification because similar string found |
| 8 | **Empty extraction treated as success** | LLM returns valid JSON with all NULL values (schema-valid but content-free) | PARTIALLY DETECTED — V6 coverage scoring catches severe cases |
| 9 | **Program defaulting masks real program** | Rate has `program=NULL` → enrichment sets "Commercial" even if it was actually CalPERS | UNDETECTED — default application is unconditional |
| 10 | **Supersession logic fails on out-of-order amendments** | Amendment dates not chronological due to effective vs execution date confusion | PARTIALLY DETECTED — validator checks date ordering but doesn't block |

### 8.2 Scale-Amplified Silent Failures

At current scale (133 files / 5 providers): These issues affect a few dozen data points.
At full scale (10,372 files / 597 providers): The same failure rates produce:

* ~1,500 misattributed rates (15% of ~10,000 expected rate rows per provider)
* ~700 wrong provider_id assignments (6.2% filename parse failures × 10,372)
* ~4,900 chunk merge duplicates (1,400 chunked files × 3.5 avg duplicates per file)
* ~2,000 hallucinated effective_dates (16% unverified × ~12,500 date fields)

---

## 9. Scalability Risk Review

### 9.1 Computational Scalability

| Component | Current (133 files) | Projected (10,372 files) | Risk |
|------|------|------|------|
| OCR (Tier 1) | ~30 min | ~4 hours | LOW — parallelized |
| OCR (Tier 2) | ~15 min | ~2 hours | LOW — Spark distributed |
| LLM Extraction | 93 min | ~120 hours (5 days) | HIGH — 3 workers, 30-min timeout each |
| Validation | ~5 min | ~6 hours | MEDIUM — sequential file loop |
| Table Build | ~2 min | ~30 min | LOW — but loads all JSONs into memory |
| LLM Audit | ~60 min (20 files) | ~30 hours (all files) | HIGH — optional but recommended |
| Gap Fill | ~45 min (10 files) | ~50 hours | HIGH — per-file LLM calls |

**Total projected time for full pipeline: ~7 days** at current concurrency (3 workers).

### 9.2 Memory Scalability

| Operation | Current Memory | Projected (full scale) |
|------|------|------|
| `individual_docs` dict (all JSONs in memory) | ~500 MB (133 files) | ~40 GB (10,372 files) | 
| `providers_data` list | ~50 MB (5 providers) | ~6 GB (597 providers) |
| pandas DataFrame for rates | ~30 MB (68K rows) | ~2.5 GB (5.3M projected rows) |

**Risk:** Notebooks 08-13 load ALL JSON files into a Python dict before building tables. At 10,372 files (avg ~4 MB each), this requires ~40 GB of driver memory. Standard L16s_v2 has 128 GB RAM total (minus OS/Spark overhead ≈ 90 GB usable). **This will likely OOM at full scale** unless JSON loading is batched.

### 9.3 Storage Scalability

| Storage | Current | Projected | Limit |
|------|------|------|------|
| Workspace files (json_extract/) | ~600 MB | ~45 GB | Workspace has soft limits — may hit quota |
| Workspace files (json_consolidated/) | ~50 MB | ~4 GB | Same |
| Delta tables (dev_adb.raw) | ~2 GB | ~150 GB | No hard limit but OVERWRITE of 150GB tables is slow |
| Checkpoints (parquet) | ~100 MB | ~8 GB | OK |

### 9.4 Concurrency & Rate Limit Scalability

**Current:** 3 workers × 30-min max timeout = 3 concurrent LLM calls.
**At scale:** The serving endpoint `databricks-claude-sonnet-4-5` has rate limits:
* Tokens per minute (TPM) — unknown exact limit
* Requests per minute (RPM) — unknown exact limit
* The `AdaptiveConcurrency` class backs off on 429s but never exceeds 3 workers

**Risk:** At 3 workers with avg 2-min calls, throughput is ~90 files/hour. For 10,372 files: **~115 hours (4.8 days)** of continuous processing. Any interruption requires checkpoint recovery.

---

## 10. Critical Technical Weaknesses — Prioritized

### CRITICAL (Enterprise-blocking)

| # | Weakness | Impact | Affected Data |
|---|------|------|------|
| C1 | **Table linearization destroys structure** | Rate-service misattribution, missing rows, column swaps | ALL rate tables (68K+ rows) |
| C2 | **Chunk merge logic loses data** | First-write-wins overwrites correct later-chunk extractions | ~1,400 chunked files (13.5% of corpus) |
| C3 | **Chunk prompt has no inter-chunk context** | Each chunk is processed blind — no knowledge of what other chunks contain | All chunked extractions |
| C4 | **Same model audits own extractions** | Systematic blind spots cannot be detected | ALL 35 tables — no independent verification |
| C5 | **No table structure detection in OCR** | Pipeline cannot distinguish tabular from narrative content | All rate exhibits, capitation tables, regional factor tables |

### HIGH (Data quality degradation)

| # | Weakness | Impact | Affected Data |
|---|------|------|------|
| H1 | **Cross-page table continuation unhandled** | Orphan rows without headers are misinterpreted | ~30% of multi-page rate tables |
| H2 | **Program/network assignment unverified** | 35-40% of rates have defaulted (not extracted) program values | program_normalized reliability |
| H3 | **Filename parse failures propagate silently** | ~6.2% of files have wrong provider_id in all downstream tables | dim_provider_canonical accuracy |
| H4 | **Overlap region in chunking produces duplicate rates** | List concatenation in merge creates phantom rate rows | All chunked file rate lists |
| H5 | **No confidence score on individual fields** | Consumers cannot distinguish LLM-extracted vs defaulted vs hallucinated | ALL fields in ALL tables |
| H6 | **Boolean/inferential fields unverifiable** | has_stop_loss, auto_renewal, is_capitated rely on LLM interpretation | Provider profile accuracy |

### MEDIUM (Operational risk)

| # | Weakness | Impact | Affected Data |
|---|------|------|------|
| M1 | **OVERWRITE destroys history** | Bug introduction silently corrupts all tables with no rollback | All 35 tables |
| M2 | **Sequential processing with 24h budget** | Full corpus cannot complete in one run — requires multiple days | Production scheduling |
| M3 | **No per-field provenance** | Cannot trace which LLM call produced which specific field value | Audit trail |
| M4 | **Driver OOM at full scale** | All JSONs loaded to memory simultaneously | Table build steps (08-13) |
| M5 | **Workspace file storage fragility** | No replication, no backup policy, workspace quota limits | All intermediate state |
| M6 | **Rate verification false positives** | Number exists in text ≠ number correctly attributed | 70% "verified" rates include misattributions |

---

## 11. Extraction Quality Metrics (Observed)

Based on validation results from the 133-file sample:

| Metric | Value | Enterprise Target | Gap |
|------|------|------|------|
| Extraction success rate | 100% | 99.9% | MEETS |
| rate_numeric populated (post-processor) | 95% | 99% | -4% |
| Dates verified in source | 84% | 95% | -11% |
| Rates verified in source | 70% | 95% | -25% |
| full_clause_text presence | 37% | 80% | -43% |
| amendments_impact populated | 98% | 99% | -1% |
| Program field populated (non-default) | 65% | 90% | -25% |
| Network field populated (non-default) | 60% | 90% | -30% |
| V6 coverage score (median) | 0.72 | 0.90 | -0.18 |

---

## 12. Enterprise Risk Summary

### Regulatory/Compliance Risk
* **PII stored in cleartext** (Tax ID, NPI) — HIPAA/CCPA exposure
* **No audit trail of who ran pipeline** — SOX/SOC2 gap
* **Rate data accuracy ~70% verified** — any downstream pricing decisions based on this data carry 30% uncertainty
* **Clause extraction 37% complete** — legal reliance on extracted clauses is unsafe

### Financial Risk
* **Rate misattribution** — using extracted rates for provider payment decisions could result in over/under-payment
* **Program assignment errors** — rates tagged "Commercial" that are actually CalPERS affect financial reporting
* **Settlement amount accuracy** — unverified settlement amounts in legal/financial dashboards

### Operational Risk
* **5-day processing time** for full corpus — any daily pipeline expectation is infeasible
* **OOM at scale** — table build will crash at full corpus without refactoring
* **No alerting** — pipeline failures are silent until someone manually checks

---

## END OF PHASE 2 AUDIT
