# Retrieval & Grounded-AI Architecture — Deep Audit (Phase 4)

*Code-grounded review of the retrieval/grounding stack: notebooks 16 (indexing), 17 (semantic clause search), 19 (hybrid router), 20 (citation answers), 23 (grounded legal QA). Citations are `notebook:line`.*

---

## Thesis

The retrieval layer is a **competent single-shot RAG demo**: one embedding model, two managed vector indexes, LLM intent-routing or query-expansion, and "stuff top-k chunks into the model and ask for `[n]` citations." It works for "find me clause language like X." It is **not** an enterprise grounded-legal-AI system, for four reasons that compound: (1) it retrieves the **wrong abstraction** — context-free OCR fragments and flattened concat strings rather than legal units with their structure intact; (2) **supersession is only half-respected and is sometimes affirmatively misreported**; (3) **"grounding" is unverified** — citations are emitted by the model but never checked against the cited text, and "confidence" is just cosine similarity; (4) there is **no precision machinery** (no reranker, no lexical/hybrid fusion, no metadata pre-filter beyond a provider name and a currency flag).

---

## How the stack is actually built

**Embeddings & indexes (16).** A single embedding model, `databricks-gte-large-en` (1024-dim), feeds two Delta Sync indexes on one endpoint (`contract_intelligence_vs_endpoint`): a **structured index** over `tbl_genie_contract_terms` (the flattened clauses+codes+parties concat from Phase 3) and a **full-text index** over OCR chunked at **1000 chars / 200 overlap** from the `step1_parsed_v2.parquet` checkpoint (`16:86-116`). One model, one similarity space, no domain tuning.

**Retrieval (17/19/20/23).** Every retrieval call is `index.similarity_search(query_text, num_results, filters)` — pure approximate-nearest-neighbor cosine top-k. **There is no reranker, no cross-encoder, no BM25/lexical channel, and no reciprocal-rank fusion anywhere in the codebase** (verified by search). Filters are limited to `provider_name` (exact metadata match) and `is_from_latest_doc="true"` on the structured index only.

**Routing & orchestration.** Three orchestration styles exist in parallel: 17 is a thin filtered search; 19 is an **LLM intent classifier → text-to-SQL or vector search**; 23 is the most advanced — **LLM query expansion (2–3 paraphrases) → multi-query retrieval → concat/dedup → chain-of-thought synthesis with multi-turn memory** (`23:209-380, 474-504`). 20 is the citation-focused RAG (`20:96-301`).

**Grounding & citation.** 20 and 23 build a numbered evidence block from the top chunks, send it to `ai_query(claude-sonnet-4-5)`, and ask for inline `[n]` markers; "used citations" are then recovered by regex (`20:289`). Citations resolve to **document filename + page** (full-text) or **filename only** (structured, `page=None` at `20:138`).

---

## Determinations

### Are answers truly grounded?
**Weakly.** The prompts do say "use ONLY the evidence" (`20:222`, `23:490`), and evidence is real retrieved text — so answers are *evidence-conditioned*. But grounding is never *verified*: nothing checks that cited source `[n]` actually supports the sentence it's attached to. The `[n]` markers are model-emitted and regex-counted, not validated by entailment. Worse, 20's synthesis prompt is tuned to **almost never abstain** — "ONLY abstain if COMPLETELY UNRELATED… you MUST provide an answer" (`20:226-234`) — so on tangential evidence the model is actively pushed to answer rather than decline. 23's prompt is more disciplined ("NEVER invent," genuine abstention path at `23:492, 498`), but still has no faithfulness check. Net: answers are grounded enough to *look* authoritative and cited, without a mechanism that guarantees the citation supports the claim. That is the most dangerous failure mode for legal AI — confident, cited, and unverified.

### Is amendment supersession respected?
**Partially, and in one place falsely.** The structured index is filtered to `is_from_latest_doc="true"` (`17:88, 19:300, 20:107, 23:256`), which is the system's only supersession mechanism in retrieval — but that flag is document-level, computed per `provider_id`, and unreliable for partial amendments and multi-contract providers (Phase 2/3 findings). The **full-text index has no currency filter at all** (`20:156, 23:293`), so it retrieves superseded OCR text with equal weight — and 20 then **hardcodes `status="CURRENT"` on every full-text citation** (`20:189`). A passage from a superseded base agreement is therefore displayed to the user as "Status: CURRENT." That is not a missing feature; it is affirmatively incorrect provenance. There is no point-in-time ("as of date D") capability and no exhibit/clause-level supersession anywhere in retrieval.

### Is retrieval enterprise-grade or prototype-grade?
**Prototype-grade.** Single embedding model with no domain/instruction tuning; no reranking or hybrid lexical+semantic retrieval (the two highest-ROI precision techniques); metadata pre-filtering limited to provider + a currency boolean; uncalibrated similarity thresholds (0.50/0.75 applied as if universal, `20:70-72`); citations at document/page granularity, not span; no retrieval-quality evaluation harness (the only "validation" tests routing accuracy and abstention on absurd off-topic queries like cryptocurrency/quantum computing, `19:587-617, 20:501-506`); and the surfaces are interactive notebooks, not served endpoints. Plus active schema-drift bugs (below).

### Where do hallucination pathways exist?
1. **False currency labeling** (`20:189`) — historical text presented as CURRENT; the user/LLM treats stale provisions as in force.
2. **Anti-abstention prompt** (`20:226-234`) — forces answers from tangential evidence.
3. **Confidence = cosine similarity** (`20:204-211, 323-338`) — "HIGH confidence" means *semantically similar*, not *correct* or *supported*. A highly-similar but contradictory or superseded passage scores HIGH. Retrieval confidence is conflated with answer confidence.
4. **No citation-faithfulness verification** — the model can attach `[2]` to a claim source 2 doesn't support and the system displays it as grounded.
5. **Text-to-SQL on a stale schema** (19) — the SQL-gen prompt describes columns that don't match the real tables (e.g. `rate_amount`/`program_name`, `dim_provider_canonical.folder_name`, `19:104, 218`); generated SQL is executed directly with errors swallowed (`19:269-275`), so a query can silently return nothing and be presented as "no rows," or the model invents plausible columns.
6. **Broken reimbursement evidence path** (`23:354`) — the rates query selects `rate_description`, which does not exist in `tbl_genie_rates_current` (real column: `rate_text`); the query errors, is caught silently (`23:377`), and the model answers rate questions from *text chunks* instead of structured rate data, with no signal that the authoritative source was missing.
7. **Context-free chunk fragments** (full-text, 1000-char windows) — a fragment severed from its section header, exhibit label, program, and effective date invites the model to fill the missing context.
8. **Silent evidence truncation** — 800 chars/citation (`20:264`), evidence block capped at 6000 chars (`23:479`), formatted top-6/top-5 only (`23:438-456`) — relevant evidence is dropped before the model sees it, with no indication.
9. **Unnormalized multi-query fusion** (`23:401-412`) — scores from different query paraphrases are concatenated and sorted directly (no normalization/RRF), producing arbitrary ranking.

---

## Does retrieval return the CORRECT abstraction of legal/business information?

**No — this is the root problem.** Each sub-question fails:

**Does chunk retrieval lose legal context?** **Yes.** The full-text index is fixed 1000-char OCR windows (`16:109`). A contract clause, a rate row, or an obligation gets split mid-structure, and the chunk is embedded with no enrichment — no provider, no contract/exhibit, no effective date, no current/historical tag attached to the embedded text. The retrieved fragment is legally context-free, so the model reconstructs context by inference.

**Does clause retrieval lose amendment context?** **Yes.** The structured index embeds `content_text` from the flattened terms table; the only amendment signal is the document-level `is_from_latest_doc` boolean. There is no amendment lineage on a clause, no "modified by Amendment 3 effective 2023-07-01," no link to the prior version it replaced. Retrieval can surface a clause but cannot tell you which amendment is responsible for it or what it superseded.

**Does reimbursement retrieval lose operational semantics?** **Yes, doubly.** Rates are retrieved (when the SQL path works) from flat `tbl_genie_rates_current` — one scalar per row, with the compound-rule collapse, fabricated program/network, and unreliable status documented in Phase 2. The **operational conditions that actually govern payment** — stop-loss attachment, outlier caps, carve-outs, lesser-of/chargemaster rules — live in *separate* tables (`tbl_contract_financial_protections`, `documents_master`) that are **not joined into retrieval**. So "what's the cardiac case rate for X" returns a number stripped of the stop-loss and carve-out logic that determines the real reimbursement. And in 23 the rate path is outright broken (`rate_description`).

**Does current-state awareness truly exist?** **No, not as state.** What exists is a *document-level latest-doc heuristic* on one index, a deduped current-rates table reachable via SQL, and a currency-blind full-text index that mislabels its results CURRENT. There is no notion of "the governing provision for facility F, program P, service S, as of date D." Current-state awareness is simulated by a flag, not reconstructed as state.

---

## Recommendations

These extend the Phase-3 target (a bitemporal contract knowledge graph + point-in-time state API). Retrieval should become an access path into that state layer, not the system of record.

**Better retrieval abstraction — retrieve legal units, not fragments.** Index **state nodes** (clause / rate-rule / obligation), each carrying `contract_id`, `exhibit`, `section`, `effective_interval`, `in_force` flag, amendment lineage, and a provenance span. Use **small-to-big / parent-document retrieval**: embed an enriched child for recall, but return the *whole clause plus its section header and definitions* as context so the model never sees a severed fragment.

**Better semantic indexing.** (1) **Structure-aware chunking** on clause/section boundaries, not 1000-char windows. (2) **Contextual embedding** — prepend a header to each chunk before embedding ("Provider X | Contract C | Exhibit C | effective 2023-07-01 | CURRENT: …") so the vector encodes who/when/where, not just text. (3) **Hybrid retrieval** — add a lexical/BM25 channel (essential for exact terms: revenue codes, CPT, dollar amounts, defined terms) and fuse with RRF. (4) **A cross-encoder reranker** over the fused candidate set for precision. (5) **Metadata pre-filtering** (provider, program, content_type, in_force, as-of date) applied *before* ANN. (6) **Calibrate** score thresholds empirically per index rather than hardcoding 0.50/0.75. (7) Evaluate with a labeled retrieval set (recall@k, nDCG), not routing-accuracy toys.

**Better grounding architecture.** (1) **Span-level citations** (character offsets into the source), not filename/page. (2) **Citation-faithfulness verification** — after generation, check each `[n]`-supported sentence against its cited span (NLI or a verifier LLM call) and drop/flag unsupported claims. (3) **Decouple confidences** — report *retrieval* confidence (calibrated relevance) separately from *answer* confidence (grounding/faithfulness coverage); never derive answer confidence from cosine similarity. (4) **Real abstention** driven by faithfulness coverage, not a prompt that discourages it. (5) Every answer surfaces **as-of date, governing amendment, and coverage** ("based on 4 of 6 amendments; Exhibit C unresolved").

**Better amendment-aware retrieval.** Attach amendment lineage to every indexed node; compute `in_force` from the event-sourced, exhibit-level supersession model (Phase 3) rather than a per-provider latest-doc flag; let users retrieve a clause's *version history* and see "modified by Amendment N." Boost in-force nodes and clearly partition historical ones — never label them CURRENT.

**Better state-aware retrieval.** Make the **point-in-time state API the first hop**: resolve `state(facility, program, service, as_of_date)` to the governing node(s), then retrieve supporting spans for explanation. Forbid currency-blind full-text from driving "as of today" answers. For reimbursement, **assemble the full rule object** (rate + stop-loss + carve-outs + lesser-of/operational conditions) before grounding, so the answer reflects what actually adjudicates a claim.

---

## What it gets right

Real evidence-conditioned answers with source pointers and an abstention concept; a sensible intent-routing idea (SQL for quantitative, vector for qualitative, hybrid for both); query expansion and multi-turn memory in 23; a currency filter on the structured index (the right instinct, even if the underlying flag is weak); and clean, extensible orchestration code. The bones of a good retrieval UX are here — what's missing is the precision machinery, verified grounding, and a state layer to retrieve *from*.

## What should evolve next (sequencing)

1. **Fix the integrity bugs now**: stop hardcoding full-text `status=CURRENT` (`20:189`); fix the `rate_description` schema drift (`23:354`) and reconcile 19's SQL-gen schema with the real tables; surface (not swallow) SQL errors.
2. **Add hybrid retrieval + a reranker** — the fastest precision win on the existing indexes.
3. **Re-chunk structure-aware and embed with contextual headers + metadata pre-filter.**
4. **Add citation-span faithfulness verification and decouple confidence from similarity.**
5. **Make retrieval state-aware** by querying the point-in-time state API first (depends on the Phase-3 amendment/state work).

Step 1 is a same-day correctness fix; steps 2–4 are independent retrieval upgrades; step 5 is the architectural inflection that turns this from "search over contracts" into "answer about what is in force."

---

*Method note: grounded in notebooks 16, 17, 19, 20, 23. The absence of reranking/lexical/RRF and the `rate_description`/`status=CURRENT` defects were verified directly against the source.*
