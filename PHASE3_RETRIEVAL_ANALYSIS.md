# PHASE 3 — Retrieval, RAG & Search Architecture Analysis
## Contract Codification Pipeline — Retrieval Intelligence Assessment
**Date:** May 19, 2026  
**Role:** Principal GenAI Retrieval Architect, Enterprise Legal AI Reviewer  
**Basis:** Verified query execution against live Delta tables + complete code inspection

---

## 1. Current Retrieval Architecture Assessment

### 1.1 What Exists Today

The system has **ZERO semantic retrieval infrastructure**. All query capability is SQL-based:

| Component | Exists? | Implementation |
|------|------|------|
| Structured SQL queries | YES | 35 Delta tables in `dev_adb.raw` |
| Keyword search (ILIKE) | YES | Available on all STRING columns |
| Column comments for Genie | YES | 67 column comments on Genie tables |
| Vector embeddings | NO | Not generated, not stored |
| Vector search index | NO | No Databricks Vector Search configured |
| Semantic similarity | NO | No embedding-based retrieval |
| RAG pipeline | NO | No retrieval-augmented generation |
| Full-text search index | NO | No inverted index, no Lucene |
| Knowledge graph | NO | No entity-relationship graph |
| Re-ranking | NO | No learned relevance scoring |
| Agentic retrieval | NO | No multi-step query planning |

### 1.2 The Three Query Interfaces

```
┌───────────────────────────────────────────────────────┐
│  INTERFACE 1: Direct SQL (Analysts, Dashboards)       │
│  • Full SQL access to all 35 tables                   │
│  • JOIN through dim_provider_canonical                 │
│  • ILIKE for text search                              │
│  • Aggregations, window functions, CTEs                │
└───────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────┐
│  INTERFACE 2: Databricks Genie (NL→SQL, planned)      │
│  • Natural language → SQL generation                   │
│  • 4 tbl_genie_* tables with column comments           │
│  • Relies on NL→SQL translation accuracy               │
│  • NO Genie Space actually configured yet              │
└───────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────┐
│  INTERFACE 3: Source PDF Access (not exposed)         │
│  • Original PDFs in Volume (read-only)                 │
│  • No user-facing PDF retrieval API                    │
│  • No citation linking (table row → PDF page)          │
└───────────────────────────────────────────────────────┘
```

---

## 2. Current Query Capability Assessment

### 2.1 Questions the System CAN Answer Reliably (SQL-Sufficient)

| Question Type | Example | Method | Reliability |
|------|------|------|------|
| **Fact lookup** | "What is the ICU per-diem rate for Garfield Medical Center?" | SQL filter on provider + service | HIGH (95%+) |
| **Aggregation** | "What's the average inpatient rate across all providers?" | SQL AVG with filters | HIGH |
| **Comparison** | "Compare stop-loss thresholds for hospitals in LA County" | SQL JOIN + ORDER BY | HIGH |
| **Existence check** | "Does St. Joseph's have a capitation arrangement?" | SQL WHERE is_capitated | HIGH |
| **Timeline** | "Show amendment history for Ukiah Valley" | SQL ORDER BY amendment_order | HIGH |
| **Count/frequency** | "How many providers have auto-renewal?" | SQL COUNT with filter | HIGH |
| **Filtering** | "All providers with rates > $10,000" | SQL WHERE rate_numeric > 10000 | HIGH |
| **Keyword clause search** | "Find termination clauses mentioning 90-day notice" | ILIKE '%90%day%notice%' | MEDIUM |
| **Program-specific rates** | "CalPERS capitation rates by provider" | SQL WHERE program_normalized | HIGH |
| **Latest document** | "Current contract terms for Sharp Memorial" | is_from_latest_doc = TRUE | HIGH |

### 2.2 Questions That Will FAIL or Produce Unreliable Results

| Question Type | Example | Why It Fails | Failure Mode |
|------|------|------|------|
| **Semantic similarity** | "Find contracts with provisions similar to Kaiser's termination clause" | No embedding comparison. ILIKE requires exact keyword match. | Returns nothing or only exact keyword matches |
| **Conceptual search** | "Which providers have the most restrictive non-compete language?" | "Restrictive" is a judgment requiring semantic understanding | SQL cannot reason about restrictiveness |
| **Cross-clause reasoning** | "If Provider A terminates, what happens to rates for existing patients?" | Answer requires joining termination + continuity of care + rate status logic | No mechanism to synthesize across multiple clause texts |
| **Comparative legal analysis** | "How does Garfield's arbitration clause compare to industry standard?" | No "industry standard" baseline, no similarity metric | Cannot compare clause meaning |
| **Temporal reasoning** | "What changed in the 2021 amendment vs the 2019 base agreement?" | `summary_of_changes` exists but is LLM-generated; no diff capability | Relies on potentially hallucinated summary |
| **Implication analysis** | "If CMS changes DRG weights, which contracts are affected?" | Requires understanding payment_type semantics + contract formula interpretation | No formula evaluation engine |
| **Negation search** | "Find contracts that do NOT have a dispute resolution clause" | Requires LEFT JOIN + IS NULL pattern (doable in SQL) but only for known topics | Silent if topic wasn't extracted |
| **Fuzzy entity resolution** | "Find all contracts with Dignity Health hospitals" | Parent-subsidiary mapping not in dim_provider_canonical | Name-based ILIKE will miss DBA/subsidiary names |
| **Open-ended exploration** | "What are the unusual or non-standard clauses in our contracts?" | Requires understanding of "standard" vs "unusual" | SQL cannot assess unusualness |
| **Citation grounding** | "Show me exactly where in the PDF this rate comes from" | source_filename exists but no page/section reference | Can identify file, cannot point to location |

### 2.3 Genie NL→SQL Limitation Analysis

Databricks Genie converts natural language to SQL. Its effectiveness depends on:

| Factor | Current State | Impact |
|------|------|------|
| Column comments | 67 comments on Genie tables (100%) | GOOD — helps Genie understand column semantics |
| Table documentation | README_tables exists but not linked to Genie | PARTIAL — Genie may not know query patterns |
| Genie Space configuration | NOT YET CREATED | BLOCKING — Genie cannot be used until space exists |
| Complex JOINs | 3 layers with different join keys | RISKY — Genie may use wrong join pattern |
| ILIKE vs semantic | Genie can generate ILIKE but not semantic queries | LIMITED — NL questions about meaning become keyword guesses |
| Ambiguous provider names | Same hospital under multiple names/IDs | RISKY — Genie may not know to use canonical resolution |

**Genie's realistic capability ceiling:** It can answer ~70% of structured fact-lookup questions reliably. For semantic questions about clause meaning, comparison, or implication, Genie's NL→SQL will either:
1. Fail to generate a query ("I don't understand")
2. Generate an ILIKE query that returns too many/few results
3. Generate a syntactically valid query that returns semantically wrong results

---

## 3. Current Semantic Intelligence Assessment

### 3.1 Semantic Content Inventory

The system stores significant unstructured text that has semantic value:

| Content | Rows | Avg Length | Max Length | Semantic Density |
|------|------|------|------|------|
| Legal clauses (verbatim) | 4,435 | 1,173 chars | 27,622 chars | VERY HIGH — every word matters legally |
| Section summaries | 17,516 | 377 chars | 4,421 chars | HIGH — condensed section content |
| Definitions | 25,156 | 108 chars | 2,348 chars | HIGH — precise legal meanings |
| Amendment summaries | 3,518 | ~200 chars | ~500 chars | MEDIUM — LLM-generated change descriptions |
| Rate formulas/notes | ~12,000 | ~100 chars | ~2,000 chars | MEDIUM — contextual rate information |

**Total semantic corpus: ~7.5 million characters of legally meaningful text** that cannot be effectively queried via SQL keyword matching alone.

### 3.2 What SQL ILIKE Can vs Cannot Find

**CAN find (keyword present in text):**
```sql
-- "90-day termination notice" → works if exact phrase exists
WHERE content_text ILIKE '%90%day%notice%'

-- "arbitration" clauses → works
WHERE content_text ILIKE '%arbitration%'
```

**CANNOT find (semantic intent without exact keywords):**
```sql
-- "most favorable renewal terms" → what keywords?
-- "clauses that protect against unilateral rate changes" → how to ILIKE this?
-- "provisions similar to force majeure" → might use different legal phrasing
-- "contracts where the provider bears more risk" → risk is distributed across multiple clauses
-- "non-standard definitions of 'Covered Services'" → requires comparing across providers
```

### 3.3 The Semantic Gap

The system has successfully **extracted** semantic content from PDFs into structured storage. But it has NOT built any **retrieval intelligence** on top of that content. The data is:

* **Stored** — YES (Delta tables, full text preserved)
* **Indexed** — NO (no inverted index, no embedding index)
* **Searchable by meaning** — NO (only by keyword)
* **Comparable** — NO (no similarity metric between clauses)
* **Reasoned over** — NO (no inference engine, no knowledge graph)

---

## 4. Major Retrieval Limitations

### 4.1 The Five Fundamental Gaps

| # | Gap | Description | Impact on Use Case |
|---|------|------|------|
| 1 | **No semantic search** | Cannot find clauses by meaning, only by keyword | "Find force majeure provisions" fails if contract uses "acts of God" |
| 2 | **No cross-clause reasoning** | Cannot synthesize answers from multiple clauses/tables | "What are the full termination consequences?" requires combining 3-5 clauses |
| 3 | **No citation grounding** | Cannot point to specific PDF location for any data point | Compliance review requires source verification — currently manual |
| 4 | **No comparative intelligence** | Cannot rank or compare clause quality/restrictiveness | "Most provider-friendly dispute resolution" is unanswerable |
| 5 | **No temporal clause evolution** | Cannot show how a specific clause changed across amendments | Amendment summaries exist but clause-level diff does not |

### 4.2 Legal Reasoning Limitations

| Legal Task | Current Capability | Gap |
|------|------|------|
| Clause compliance check | Manual review of extracted text | No automated compliance rules engine |
| Risk scoring | None | No risk taxonomy, no scoring model |
| Obligation identification | Section summaries (LLM-generated) | No structured obligation extraction |
| Deadline tracking | effective_date + expiration_date | No renewal deadline alerting |
| Conflict detection | None | Cannot detect contradictory clauses across amendments |
| Regulatory change impact | None | Cannot map regulatory requirements to affected clauses |
| Precedent search | ILIKE on clause text | Cannot find semantically similar precedents |

---

## 5. SQL vs Semantic Retrieval Analysis

### 5.1 The 70/30 Split

Based on the data characteristics and expected query patterns:

**~70% of high-value business questions are SQL-answerable:**
* Rate lookups, comparisons, aggregations
* Provider profiles, counts, existence checks
* Timeline/history queries with explicit filters
* Document inventory and classification
* Financial protection thresholds

**~30% of high-value business questions require semantic retrieval:**
* Clause comparison and similarity
* Open-ended legal exploration
* Contextual Q&A requiring synthesis
* Non-standard provision discovery
* Cross-reference and implication analysis

### 5.2 The Critical Insight

**The current system is excellently optimized for the 70%.** The three-layer table architecture, identity resolution, amendment dedup, and program normalization are sophisticated SQL-layer engineering. Genie integration (once configured) will make this accessible via natural language.

**The 30% gap is not a SQL problem — it's a fundamentally different retrieval paradigm.** No amount of additional SQL tables, views, or Genie tuning will enable:
* Finding semantically similar clauses
* Answering "why" and "how" questions about contract intent
* Synthesizing answers from multiple document sections
* Grounding responses with source citations

---

## 6. RAG Necessity Assessment

### 6.1 Is RAG Actually Needed?

**YES — but ONLY for the 30% semantic gap.** The assessment is nuanced:

| Use Case | RAG Needed? | Rationale |
|------|------|------|
| Rate lookups | NO | SQL is faster, more accurate, and deterministic |
| Provider comparisons | NO | Structured aggregation is superior |
| Clause semantic search | YES | Embedding-based retrieval is the only viable path |
| Contextual Q&A | YES | LLM reasoning over retrieved clauses |
| Cross-document synthesis | YES | Multi-chunk retrieval + LLM synthesis |
| Historical clause evolution | MAYBE | Could use SQL + LLM for diff, but RAG helps discovery |
| Compliance checking | YES | Requires retrieving relevant clauses for a given regulation |
| Risk assessment | YES | Requires semantic understanding of clause implications |

### 6.2 What Kind of RAG?

**NOT basic RAG.** The contract domain demands:

| RAG Approach | Appropriate? | Reasoning |
|------|------|------|
| Naive RAG (chunk → embed → retrieve → generate) | NO | Legal text requires precise boundaries; naive chunking destroys legal meaning |
| Clause-level RAG | YES | Each clause is a semantically complete unit (avg 1,173 chars — perfect embedding size) |
| Hybrid SQL + RAG | YES | Structured queries narrow the search space, then semantic retrieval finds specific clauses |
| Hierarchical RAG | MAYBE | Document → Section → Clause hierarchy could improve precision |
| GraphRAG | MAYBE | Amendment chains and exhibit references form a natural graph, but complexity may not justify ROI |
| Agentic RAG | YES (later) | Complex legal questions require multi-step retrieval planning |

### 6.3 Why Clause-Level Indexing Is the Natural Fit

The system already has the perfect RAG corpus pre-built:

```
tbl_genie_contract_terms:
  • 4,435 clauses (avg 1,173 chars) — semantically complete units
  • 25,156 definitions (avg 108 chars) — precise meanings
  • 17,516 sections (avg 377 chars) — content summaries
  • Each row already has: provider_name, topic, content_type, is_from_latest_doc
```

This is **NOT a raw-PDF-chunking problem**. The hard work of extracting semantically coherent units has ALREADY been done. The remaining gap is purely the retrieval index on top of it.

---

## 7. Hybrid Retrieval Necessity Assessment

### 7.1 The Hybrid Architecture That Fits

```
┌──────────────────────────────────────────────────────────────────────┐
│                    USER QUESTION                                      │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
                    ┌───────▼───────┐
                    │  QUERY ROUTER  │  ← Classifies: structured vs semantic
                    └───┬───────┬───┘
                        │           │
           ┌───────────▼─┐   ┌────▼────────────┐
           │  SQL PATH     │   │  SEMANTIC PATH   │
           │  (70% Qs)     │   │  (30% Qs)        │
           │               │   │                  │
           │ Genie NL→SQL  │   │ 1. Metadata      │
           │ or direct SQL │   │    filter (SQL)   │
           │               │   │ 2. Vector search  │
           │ Rates, dates, │   │    (embeddings)  │
           │ counts, aggs  │   │ 3. LLM synthesis  │
           │               │   │    + citation     │
           └───────┬───────┘   └────┬─────────────┘
                    │                   │
                    └───────┬─────────┘
                            │
                    ┌───────▼───────┐
                    │   RESPONSE +   │
                    │   CITATIONS    │
                    └───────────────┘
```

### 7.2 Why Hybrid (Not Pure RAG)

| Reason | Detail |
|------|------|
| **Rates are structured** | RAG would be slower and less accurate than SQL for rate lookups |
| **Identity resolution is SQL** | dim_provider_canonical is a deterministic mapping — no need for fuzzy retrieval |
| **Metadata filtering narrows search** | Before semantic search, filter by provider/doc_type/topic via SQL |
| **Aggregations are impossible with RAG** | "Average rate across 278 providers" requires SQL, not generation |
| **Legal precision demands structured constraints** | "Current rates only" = SQL filter, not semantic interpretation |

---

## 8. Trust & Grounding Assessment

### 8.1 Current Grounding Capabilities

| Trust Dimension | Current State | Enterprise Target |
|------|------|------|
| **Source traceability** | `source_filename` on every row | Need: page/section-level citation |
| **Answer provenance** | None — no mechanism to show WHERE data came from | Need: clickable citation to PDF |
| **Confidence scoring** | v6_coverage_pct per document, no per-field scores | Need: per-answer confidence |
| **Hallucination guardrails** | Temperature 0 + date/rate verification (extraction-time only) | Need: query-time grounding checks |
| **Explainability** | None — answers have no reasoning trail | Need: "I found this in Section 5.2 of Amendment 3" |

### 8.2 The Enterprise Trust Gap

For a legal AI system to be enterprise-trustworthy, every answer must provide:

1. **The answer itself** (currently: SQL result or clause text)
2. **The source evidence** (currently: filename only, no page/section)
3. **The reasoning path** (currently: none)
4. **A confidence indicator** (currently: none at query time)
5. **An escape hatch** (currently: none — no "I'm not sure" mechanism)

**Current trust level: INSUFFICIENT for legal reliance.** An attorney reviewing extracted clause text cannot:
* Verify it's complete (might be truncated)
* Verify attribution (might be from wrong exhibit)
* See surrounding context (adjacent clauses that modify meaning)
* Confirm recency (might be from superseded amendment)

### 8.3 What RAG Enables for Trust

A properly implemented RAG layer adds:

| Trust Feature | How RAG Provides It |
|------|------|
| Source citation | Retrieved chunk includes document ID + section + page (if indexed) |
| Context window | Retrieves surrounding clauses alongside the exact match |
| Confidence signal | Embedding distance score indicates match quality |
| Multi-source corroboration | Can show same concept from multiple documents |
| Freshness guarantee | Metadata filter ensures only latest-doc results |
| Refusal capability | Low similarity scores trigger "insufficient evidence" response |

---

## 9. Recommended Retrieval Direction

### 9.1 What to Build (Prioritized)

| Priority | Component | Justification | Complexity |
|------|------|------|------|
| **P1** | Genie Space configuration | Unlocks 70% of queries with zero new infrastructure | LOW (hours) |
| **P2** | Vector index on clause/section/definition text | Enables semantic search on 47K existing text units | MEDIUM (days) |
| **P3** | Hybrid query router | Directs structured Qs to SQL, semantic Qs to vector search | MEDIUM |
| **P4** | Citation linking (row → PDF page) | Page-level source attribution for compliance | MEDIUM |
| **P5** | LLM synthesis layer | Generates grounded answers from retrieved clauses | MEDIUM-HIGH |
| **P6** | Comparative clause intelligence | Cross-provider similarity scoring | HIGH |
| **P7** | Agentic multi-step retrieval | Complex legal questions requiring planning | HIGH |

### 9.2 What NOT to Build (Anti-Recommendations)

| Technology | Why NOT to Build It |
|------|------|
| **Full GraphRAG** | The relationship structure (provider → contract → amendment → clause) is already modeled in SQL tables. A knowledge graph adds complexity without clear value beyond what JOINs provide. |
| **Re-ranking models** | The corpus is small enough (47K text units) that brute-force vector search with metadata filtering will be fast enough. Re-ranking adds latency without meaningful precision improvement at this scale. |
| **Custom embedding fine-tuning** | General-purpose legal embeddings (e.g., `databricks-gte-large-en`) perform adequately on contract language. Fine-tuning requires labeled data that doesn't exist. |
| **Document-level embedding** | Documents are too long (50-200 pages) for single embeddings. The clause-level extraction already solves the chunking problem. |
| **Multi-modal retrieval** | Rate tables as images might help OCR quality, but the extraction pipeline already handles this. Multi-modal retrieval at query time adds complexity without clear benefit. |

### 9.3 The Minimal Viable Semantic Layer

The fastest path to enterprise-grade retrieval:

```
1. EMBED: tbl_genie_contract_terms.content_text
   • 47,107 rows (clauses + sections + definitions)
   • Model: databricks-gte-large-en (1024 dims)
   • Store in Databricks Vector Search (Delta Sync index)
   • Metadata columns: provider_name, content_type, topic, is_from_latest_doc

2. QUERY: Hybrid retrieval function
   • Input: user question (natural language)
   • Step 1: Metadata filter (provider, topic, recency)
   • Step 2: Vector similarity search (top-k=10)
   • Step 3: LLM synthesizes answer from retrieved chunks
   • Step 4: Return answer + source citations

3. GROUND: Every answer includes
   • Source filename(s)
   • Content type and topic
   • Similarity score (confidence proxy)
   • "Based on [document] Section [X]" citation format
```

---

## 10. Enterprise Legal AI Comparison

### 10.1 Current System vs Enterprise Legal AI Standards

| Capability | Current System | Enterprise Standard (e.g., Harvey, Luminance, Kira) | Gap Severity |
|------|------|------|------|
| Structured data extraction | EXCELLENT | Comparable | None |
| Rate table extraction | GOOD (with known limitations) | Typically uses layout-aware OCR | Medium |
| Clause retrieval by keyword | BASIC | Full semantic + keyword hybrid | High |
| Clause comparison | NONE | Cross-document similarity, deviation scoring | Critical |
| Risk scoring | NONE | Automated risk flagging per clause | Critical |
| Citation grounding | Filename only | Page, paragraph, highlighted text | High |
| Change detection (diff) | Amendment summary (LLM-generated) | Token-level clause diff with visual highlight | High |
| Regulatory mapping | NONE | Clause-to-regulation linking | High |
| Confidence scoring | Document-level only | Per-answer, per-field confidence | High |
| Multi-user collaboration | NONE | Annotation, approval workflows, comments | Medium |
| Audit trail | source_filename | Full provenance: who extracted, when, which model version | Medium |

### 10.2 Competitive Position Assessment

**What this system does BETTER than commercial legal AI:**
* Rate extraction at scale (10K+ contracts) — most commercial tools are single-document
* Amendment chain resolution and rate supersession — unique domain logic
* Identity resolution across provider IDs — custom to BSC's data landscape
* SQL-queryable structured output — most legal AI returns documents, not tables

**What commercial legal AI does better:**
* Semantic clause search and comparison
* Source citation with highlighted text
* Per-field confidence scoring
* Regulatory compliance mapping
* Collaborative review workflows

---

## 11. Future Scalability Concerns

### 11.1 Retrieval Scale Projections

| Metric | Current (5 providers) | Full (597 providers) | 3-Year Projection |
|------|------|------|------|
| Clause text rows | 4,435 | ~350,000 | ~500,000 |
| Definition rows | 25,156 | ~2,000,000 | ~3,000,000 |
| Section rows | 17,516 | ~1,400,000 | ~2,000,000 |
| Total embeddable units | 47,107 | ~3,750,000 | ~5,500,000 |
| Vector index size (1024 dims) | ~180 MB | ~14 GB | ~21 GB |

Databricks Vector Search supports billions of vectors — this scale is easily handled.

### 11.2 Query Latency Projections

| Query Type | Current (SQL) | With Vector Search (projected) |
|------|------|------|
| Rate lookup | <1s | <1s (unchanged — still SQL) |
| Keyword clause search (ILIKE) | 2-5s at full scale | <500ms (vector search) |
| Semantic clause search | N/A (not possible) | <2s (embed query + ANN search + LLM) |
| Cross-provider comparison | 5-10s (SQL aggregation) | 3-5s (vector search + LLM) |
| Multi-step legal question | N/A (not possible) | 5-15s (agentic: multiple retrievals + synthesis) |

### 11.3 Cost Projections

| Component | One-Time Cost | Monthly Cost |
|------|------|------|
| Embedding generation (3.75M rows) | ~$50-100 (embedding API) | $0 (one-time) |
| Vector Search endpoint (serverless) | $0 | ~$200-500/month (depending on QPS) |
| LLM synthesis calls (query-time) | $0 | ~$500-2000/month (depending on query volume) |
| Delta Sync index maintenance | $0 | Included in Vector Search cost |

---

## 12. Final Assessment: The Retrieval Architecture Verdict

### 12.1 The Current System Is a FOUNDATION, Not a Destination

The pipeline has accomplished the hardest part of legal AI: **extracting structured, queryable data from unstructured PDFs at scale.** This is genuinely impressive engineering. But it stops at the extraction boundary.

**The system is currently a data warehouse, not an intelligence layer.**

It stores answers but cannot discover them. It preserves legal language but cannot reason about it. It knows what's in contracts but cannot tell you what it means.

### 12.2 The Three-Phase Retrieval Roadmap (Recommended)

**Phase A (Weeks 1-2): Activate SQL Intelligence**
* Configure Genie Space with 4 tbl_genie_* tables
* Add table-level documentation and sample questions
* Verify NL→SQL accuracy on 50 test questions
* This unlocks 70% of business value with zero new infrastructure

**Phase B (Weeks 3-6): Add Semantic Retrieval**
* Generate embeddings for tbl_genie_contract_terms (47K rows)
* Create Databricks Vector Search index with metadata filters
* Build hybrid query function (SQL for structured, vector for semantic)
* Add citation formatting (source_filename + content_type + topic)
* This unlocks the 30% semantic gap

**Phase C (Weeks 7-12): Enterprise Intelligence Layer**
* Agentic retrieval for complex multi-step questions
* Cross-provider clause comparison scoring
* Per-answer confidence scoring based on retrieval distance
* Regulatory compliance mapping (clause → regulation)
* Change detection (clause diff across amendment versions)

### 12.3 The Bottom Line

**RAG is needed — but only for semantic retrieval on clause text.** The structured data (rates, dates, parties, financials) should remain SQL-only. The optimal architecture is a **hybrid router** that sends structured questions to Genie/SQL and semantic questions to a clause-level vector index with LLM synthesis.

The system does NOT need:
* Full document re-embedding (clauses are already extracted)
* Knowledge graphs (SQL relationships are sufficient)
* Custom embedding models (general-purpose legal embeddings work)
* Re-ranking (corpus is small enough for brute-force at scale)

The system DOES need:
* Genie Space (immediate, zero-cost activation of SQL intelligence)
* Vector index on 47K clause/section/definition rows
* Hybrid router (structured vs semantic query classification)
* Citation formatting (already has source_filename — needs presentation layer)
* Confidence scoring (embedding distance as trust proxy)

---

## END OF PHASE 3 ANALYSIS
