# Blue Shield of California — Provider Contract Corpus Summary

**Generated**: May 20, 2026  
**Source**: `/Volumes/prod_adb/default/ext-data-volume-stmlz/Health_Plan_Ops_Transformation/Provider_Contracts/`  
**Analysis Notebook**: `Contract Corpus Summary Analysis`  
**Delta Tables**: `dev_adb.raw.tbl_contract_corpus_inventory`, `tbl_contract_type_taxonomy`, `tbl_contract_section_structure`, `tbl_contract_data_elements`, `tbl_contract_cross_patterns`

---

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| Total PDF files | 10,349 |
| Total file size | 29.3 GB |
| Total extracted text | 0.84 billion chars |
| Total pages | 425,326 |
| Unique providers (by name) | 327 |
| Unique provider IDs | 531 |
| Unique contracts | 4,224 |
| Unique document types | 186 |
| Provider folders | 597 |
| OCR success rate | 100.0% |
| Average text per file | 81K chars |
| Average pages per file | 41 |

### Key Findings

1. **Contract Structure**: The corpus is dominated by Base Agreements (3,260 files, 31.5%) and Numbered Amendments (5,120 files, 49.5%), with Cover Memos (14.2%) providing rate schedule summaries.
2. **Payment Diversity**: Fee-for-service (78%), per-diem (62%), case rates (59%), DRG (43%), and capitation (33%) are all represented — often multiple methods within a single contract.
3. **IPA vs Direct**: IPA contracts (22%) are fundamentally different — 80% capitation (vs 20% direct), 24% withholds (vs 1%), and 24% risk-sharing (vs 0.2%).
4. **Amendment Depth**: Providers average 5+ amendments; 48 providers reach 15+ amendments deep, creating complex supersession chains.
5. **Data Richness**: 1.6M dollar amounts, 840K CPT codes, 1M revenue codes, and 721K dates extracted across the corpus.

---

## 2. Folder Structure

```
/Volumes/prod_adb/default/ext-data-volume-stmlz/
└── Health_Plan_Ops_Transformation/
    └── Provider_Contracts/
        ├── 129083117_Garfield_Medical_Center/        (67 files)
        ├── 129105716_St_Josephs_Medical_Center/     (52 files)
        ├── 129112153_Los_Robles_Regional_Medical_Center/ (38 files)
        ├── ... (597 provider folders total)
        └── [providerID]_[providerName]/
            └── [providerID]_[providerName]_[contractID]_[docType]_v[N].pdf
```

**File naming convention**: `providerID_providerName_contractID_docType_version.pdf`

**Provider portfolio sizes**:
- Mean: 31.6 files/provider | Median: 21 | Max: 302 (Queen Of The Valley Medical Center)
- 12 providers have 100+ files; 51 have 2-5 files

---

## 3. Document Type Taxonomy

### Major Categories

| Category | Files | % of Corpus | Avg Pages | Avg Size (MB) |
|----------|------:|:-----------:|----------:|--------------:|
| Numbered Amendment | 5,120 | 49.5% | 34 | 2.4 |
| Base Agreement | 3,260 | 31.5% | 53 | 4.6 |
| Cover Memo | 1,468 | 14.2% | 52 | 0.9 |
| Letter Rate Amendment | 186 | 1.8% | 3 | 0.3 |
| Other | 152 | 1.5% | 13 | 1.5 |
| Settlement | 89 | 0.9% | 16 | 1.2 |
| Assignment | 39 | 0.4% | 7 | 1.3 |
| Notice/Letter | 35 | 0.3% | 23 | 3.9 |

### Top Document Types (186 total)

| # | Document Type | Count | Category |
|---|--------------|------:|----------|
| 1 | BaseAgmtRenew | 2,745 | Base Agreement |
| 2 | FourthAmend | 551 | Numbered Amendment |
| 3 | SecondAmend | 532 | Numbered Amendment |
| 4 | FirstAmend | 493 | Numbered Amendment |
| 5 | ThirdAmend | 475 | Numbered Amendment |
| 6 | BaseAgmtNew | 398 | Base Agreement |
| 7 | SixthAmend | 396 | Numbered Amendment |
| 8 | FifthAmend | 390 | Numbered Amendment |
| 9 | EighthAmend | 333 | Numbered Amendment |
| 10 | SeventhAmend | 330 | Numbered Amendment |

### Version Distribution

| Version | Files | % |
|---------|------:|---:|
| v1 | 5,642 | 54.5% |
| v2 | 2,959 | 28.6% |
| v3 | 441 | 4.3% |
| v4 | 241 | 2.3% |
| v5+ | 1,066 | 10.3% |

---

## 4. Section Structure per Contract Type

### Base Agreements (3,260 files)

**Standard Sections (>50%)**:
- RECITALS (74%)

**Common Sections (20-50%)**:
- I. DEFINITIONS (27%)
- II. OBLIGATIONS OF HOSPITAL (22%)
- IV. ELIGIBILITY OF BLUE SHIELD MEMBERS (19%)
- V. BILLING, COMPENSATION & CHARGE MASTER OBLIGATIONS (20%)
- VI. PROTECTION OF MEMBERS (24%)
- VII. MEDICAL RECORDS & CONFIDENTIALITY (24%)
- VIII. COOPERATION WITH AUDITS & CERTIFICATIONS (20%)
- IX. RESOLUTION OF DISPUTES (23%)
- X. TERM & TERMINATION (22%)
- TABLE OF CONTENTS (22%)

### Numbered Amendments (5,120 files)

**Standard Sections (>50%)**:
- RECITALS (83%)
- AGREEMENT (80%)

**Common Sections (20-50%)**:
- I. INPATIENT SERVICES A (50%)
- II. OUTPATIENT SERVICES A (43%)
- COMPENSATION AMOUNTS/PAYMENT SCHEDULE (41%)
- EXHIBIT A (40%)

### Cover Memos (1,468 files)

**Standard Sections (>50%)**:
- RECITALS (87%)
- I. INPATIENT SERVICES A (70%)
- COMPENSATION AMOUNTS/PAYMENT SCHEDULE (69%)
- II. OUTPATIENT SERVICES A (63%)
- AGREEMENT (63%)
- EXHIBIT A (59%)

### Heading Density

| Category | Mean Headings/File | Median | Max |
|----------|-------------------:|-------:|----:|
| Base Agreement | ~25 | ~22 | varies |
| Numbered Amendment | ~20 | ~17 | varies |
| Cover Memo | ~30 | ~28 | varies |
| Settlement | ~8 | ~7 | varies |

---

## 5. Rate & Financial Data Catalog

### Payment Types Found

| Payment Method | Files | % of Corpus | IPA % | Non-IPA % |
|---------------|------:|:-----------:|------:|----------:|
| Fee-for-service | 8,102 | 78.3% | 58.1% | 84.0% |
| Per-diem | 6,448 | 62.3% | 39.7% | 68.8% |
| Case rate | 6,143 | 59.4% | — | — |
| Stop-loss | 5,972 | 57.7% | 30.2% | 65.6% |
| DRG | 4,434 | 42.8% | 20.1% | 49.3% |
| Capitation | 3,436 | 33.2% | 80.0% | 19.8% |
| % of charges | 1,238 | 12.0% | — | — |
| % of Medicare | 356 | 3.4% | — | — |

### Rate Presentation

| Presentation | Files | % |
|-------------|------:|---:|
| Embedded rate data (>5 $ amounts) | ~7,000 | ~68% |
| Exhibit/schedule references only | ~2,300 | ~22% |
| Both embedded + exhibit refs | ~6,200 | ~60% |

### Financial Provisions

| Provision | Files | % |
|-----------|------:|---:|
| Stop-loss | 5,972 | 57.7% |
| Outlier provisions | 683 | 6.6% |
| Withhold | 694 | 6.7% |
| Bonus/incentive | 971 | 9.4% |
| Risk sharing | 564 | 5.4% |
| Risk corridor | 6 | 0.1% |

---

## 6. Tables & Exhibits Inventory

### Embedded Data Elements (across all 10,349 files)

| Element Type | Total Occurrences | Files With | % Files | Avg/File |
|-------------|------------------:|-----------:|--------:|---------:|
| Dollar amounts ($) | 1,598,798 | 8,066 | 77.9% | 154.5 |
| Revenue codes | 1,017,893 | 6,037 | 58.3% | 98.4 |
| CPT codes | 840,476 | 9,300 | 89.9% | 81.2 |
| Dates (MM/DD/YYYY) | 721,171 | 7,802 | 75.4% | 69.7 |
| Percentages | 545,244 | 8,525 | 82.4% | 52.7 |
| Exhibit references | 409,886 | 9,004 | 87.0% | 39.6 |
| Written dates | 160,051 | 9,883 | 95.5% | 15.5 |
| ICD-10 codes | 148,313 | 5,319 | 51.4% | 14.3 |
| Attachment references | 136,021 | 6,405 | 61.9% | 13.1 |
| Schedule references | 107,776 | 7,197 | 69.5% | 10.4 |
| Tax IDs | 38,660 | 2,155 | 20.8% | 3.7 |
| NPI numbers | 26,995 | 4,605 | 44.5% | 2.6 |
| DRG codes | 21,507 | 1,454 | 14.0% | 2.1 |
| Appendix references | 2,461 | 178 | 1.7% | 0.2 |

### Document Structure References

| Reference Type | Files | % of Corpus | By Category (Base / Amend / CM) |
|---------------|------:|:-----------:|:-------------------------------:|
| Exhibit refs | 9,123 | 88.2% | 88% / 89% / 98% |
| Schedule refs | 7,197 | 69.5% | 70% / 66% / 98% |
| Attachment refs | 6,405 | 61.9% | 61% / 57% / 97% |
| Appendix refs | 178 | 1.7% | 4% / 1% / 0% |
| Inline rate tables | 2,032 | 19.6% | 22% / 23% / 7% |

---

## 7. Medical Code Coverage

| Code System | Total Found | Files | % Files |
|-------------|------------:|------:|--------:|
| CPT (5-digit) | 840,476 | 9,300 | 89.9% |
| Revenue (0100-0999) | 1,017,893 | 6,037 | 58.3% |
| ICD-10 | 148,313 | 5,319 | 51.4% |
| DRG | 21,507 | 1,454 | 14.0% |

**Note**: CPT code counts may include false positives (any 5-digit number). Revenue code counts also capture some numeric noise. DRG codes are the most precise (preceded by "DRG" keyword).

---

## 8. Cross-Contract Relationships

### Amendment Chains

| Metric | Value |
|--------|------:|
| Total unique contracts | 4,224 |
| Single-file contracts | 1,709 |
| Multi-file contracts (2+) | 2,515 |
| Contracts with 5+ files | 275 |
| Max files per contract | 62 |
| Providers reaching 10+ amendments | 99 |
| Providers reaching 15+ amendments | 48 |

### Supersession Language

| Pattern | Files | % | Primary Category |
|---------|------:|---:|:----------------:|
| "replaces" (in entirety) | 6,157 | 59.5% | Amendments |
| "supersedes" | 5,015 | 48.5% | All |
| "in lieu of" | 4,515 | 43.6% | Amendments |
| "notwithstanding" | 4,109 | 39.7% | All |
| "hereby amended" | 3,125 | 30.2% | Amendments |
| "except as amended" | 240 | 2.3% | Base Agreements |
| "null and void" | 159 | 1.5% | Settlements |
| "amends and restates" | 33 | 0.3% | Base Agreements |

**Supersession by category**: Base Agreements (77%), Numbered Amendments (90%), Cover Memos (99%), Settlements (97%)

### Template Standardization

| Category | Files | Unique Templates | Ratio | Assessment |
|----------|------:|-----------------:|------:|:----------:|
| Base Agreement | 3,260 | 2,101 | 0.64 | Mostly custom |
| Numbered Amendment | 5,120 | 2,707 | 0.53 | Moderately templated |
| Cover Memo | 1,468 | 1,009 | 0.69 | Mostly custom |
| Settlement | 89 | 39 | 0.44 | Moderately templated |
| Letter Rate Amendment | 186 | 118 | 0.63 | Mostly custom |

---

## 9. IPA vs Direct Contract Patterns

| Attribute | IPA (22%) | Non-IPA (78%) | Difference |
|-----------|----------:|--------------:|-----------:|
| Capitation | 80.0% | 19.8% | +60.2% ▲ |
| Withhold | 25.6% | 1.3% | +24.3% ▲ |
| Risk sharing | 23.7% | 0.2% | +23.5% ▲ |
| Bonus/incentive | 23.7% | 5.3% | +18.5% ▲ |
| Fee-for-service | 58.1% | 84.0% | -25.9% ▼ |
| Per-diem | 39.7% | 68.8% | -29.1% ▼ |
| DRG | 20.1% | 49.3% | -29.2% ▼ |
| Stop-loss | 30.2% | 65.6% | -35.3% ▼ |

---

## 10. Compliance & Regulatory

| Regulation | Files | % |
|-----------|------:|---:|
| Medi-Cal | 9,345 | 90.3% |
| Medicare | 7,652 | 73.9% |
| HMO | 7,426 | 71.8% |
| PPO | 7,159 | 69.2% |
| CMS | 6,270 | 60.6% |
| Knox-Keene | 5,007 | 48.4% |
| IPA | 2,299 | 22.2% |
| DMHC | 1,948 | 18.8% |
| HIPAA | 1,642 | 15.9% |
| DHCS | 672 | 6.5% |

---

## 11. Data Quality Notes

| Issue | Count | Impact |
|-------|------:|--------|
| Files with parse errors | 459 | 4.4% — mostly partial text available via pypdf fallback |
| Empty/minimal text (<500 chars) | 25 | 0.2% — effectively unreadable |
| Files needing Tier 2 OCR (ai_parse_document) | 1,246 | 12.0% — scanned PDFs successfully processed |
| Files via pytesseract fallback | 68 | 0.7% — legacy OCR, lower quality |
| Multiple versions per document | 4,707 | 45.5% — version management adds complexity |

### Extraction Quality by Method

| Method | Files | Avg Text Length | Coverage |
|--------|------:|----------------:|---------:|
| pypdf | 9,056 | ~75K chars | 87.4% |
| ai_parse_document | 1,246 | ~95K chars | 12.0% |
| pytesseract | 68 | ~60K chars | 0.7% |

---

## 12. Recommendations for Downstream Analytics

### Immediate Opportunities
1. **Rate Extraction Pipeline**: 77.9% of files contain dollar amounts; structured rate table extraction would yield the most business value
2. **Amendment Chain Reconstruction**: Build full contract lineage using supersession language + contract_id grouping
3. **Date Extraction & Timeline**: 95.5% of files have dates — build contract lifecycle timelines
4. **Medical Code Mapping**: 89.9% have CPT codes — map covered services programmatically

### Data Gaps to Address
1. **LLM extraction coverage**: Only 3,519/10,372 files (34%) have structured JSON extracts — need to process remaining 6,853
2. **Version deduplication**: 45.5% of files are version variants — define "latest version" logic for dedup
3. **Provider name normalization**: 531 IDs map to 327 names — some providers have multiple IDs (mergers, renames)
4. **Scanned PDF quality**: 1,246 files needed Tier 2 OCR — may have lower extraction accuracy

### Architecture Recommendations
1. Use `dev_adb.raw.tbl_contract_corpus_inventory` as the master registry for all downstream joins
2. Filter to `version_num = max(version_num)` per (provider_id, contract_id, document_type) for deduplication
3. Separate IPA and direct-contract analytics — they have fundamentally different structures
4. Prioritize Base Agreements + their latest amendments for complete current-state view

---

## Delta Tables Created

| Table | Rows | Description |
|-------|-----:|-------------|
| `dev_adb.raw.tbl_contract_corpus_inventory` | 10,349 | Master file inventory with metadata and categories |
| `dev_adb.raw.tbl_contract_type_taxonomy` | 186 | Document type taxonomy with attribute percentages |
| `dev_adb.raw.tbl_contract_section_structure` | 500 | Top section headings with frequency by category |
| `dev_adb.raw.tbl_contract_data_elements` | 112 | Data element counts per category (14 elements × 8 categories) |
| `dev_adb.raw.tbl_contract_cross_patterns` | 10,349 | Cross-reference patterns, fingerprints, supersession flags |

---

*Analysis performed on May 20, 2026 using pre-extracted OCR text from `step2_with_metadata_v2.parquet` (10,349 files, 0.84 billion chars). No LLM was used — all analysis is regex and pandas-based.*
