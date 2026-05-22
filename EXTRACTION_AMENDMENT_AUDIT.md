# Extraction Architecture & Amendment Intelligence — Deep Audit (Phase 2)

*Independent, code-grounded review of notebooks 02–12 (plus 16's chunker). Citations are `notebook:line`. This is my own reconstruction from the source, not a summary of the repo's existing `PHASE*.md` documents.*

---

## 1. How the extraction pipeline actually works

The path from PDF to relational fact is: **OCR (02) → filename metadata (03) → LLM extraction against a fixed schema (04/05) → JSON validation in place (06) → per-provider consolidation (07) → base Delta tables (08–10) → enrichment (11) → Genie/serving current-state tables (12/13)**. The legal and financial "truth" of a contract is decided in three places that matter most: the **prompt + schema** (04/05), the **consolidation supersession engine** (07), and the **document-level status fill** (08). Everything downstream inherits the semantics those three steps impose.

### OCR strategy (02)
Two tiers: `pypdf` in a `ProcessPoolExecutor` for ~82% of files (`02:74-110`), then `ai_parse_document` for the scanned/low-yield remainder (`02:127-201`). The routing gate is purely **text-volume heuristics** — `MIN_GOOD_CHARS` and a `text_length / num_pages` ratio (`02:101-103`). The Tier-1/Tier-2 reconciliation picks whichever output is **longer** (`02:173-189`).

This is a reasonable, scalable ingestion design, but it is **layout-blind**. `ai_parse_document` output is flattened with `concat_ws('\n\n', transform(... elements ... e:content))` (`02:160`) — every structural element (including tables) is concatenated into a single text blob with no table boundaries, no column/row association, no reading-order guarantee. Provider rate schedules are **tables**. Collapsing a per-diem grid or a capitation matrix into linear text is the first and most consequential loss in the pipeline, because every rate the LLM later "reads" comes from a delinearized table. Picking the *longer* of two extractions is also a weak quality proxy — a garbled-but-long OCR pass can beat a clean-but-short `pypdf` pass.

### Metadata parsing (03) — the filename is load-bearing
`provider_id`, `provider_name`, `contract_id`, `document_type`, and `version` are parsed from the **filename** with one regex (`03:38`), with a positional `split('_')` fallback (`03:42-46`). Nothing here reads the document body.

This is the single most underappreciated dependency in the system. **Document type drives prompt selection** (04:243-280), **amendment ordering** (07:127-149), and **document-role classification** (07:151-166). All of that is bootstrapped from a filename string. A misnamed, renamed, or non-conforming file silently falls to the fallback branch → `document_type = None` → routed to the *base-agreement* prompt even if it is an amendment → its `amendments_impact` is never properly elicited and its amendment order collapses to a sentinel. There is no content-based document classifier anywhere to catch this.

### Chunking + merge (05) — where large contracts lose coherence
Files over 150K chars are split into 80K windows with 2K overlap, broken on separators (`05:507-570`). Each chunk is sent to the LLM **against the full schema** with a generic (non-document-type-specific) prompt (`05:576-582`), then `_merge_extractions` recombines them (`05:615-632`). The merge rule is:

- **Scalars / dicts: first-non-empty wins.** Whatever chunk 1 said about `contract_overview`, `payment_type`, `amendments_impact` is frozen; later chunks cannot correct it (`05:624`).
- **Lists: blind `extend()`** — every chunk's rate rows are concatenated with **no dedup** (`05:626-627`). The 2K overlap region is extracted twice, so rates straddling a boundary are **duplicated by construction**.

So for the largest, most complex contracts (exactly the ones with the richest rate tables), the merge step both **drops corrective information** (scalar fields) and **injects duplicate rates** (lists). The chunk prompt also asks each chunk to populate the entire schema, inviting each chunk to emit half-seen sections.

### Prompts + JSON schema (04/05) — strong structure, dangerous defaults
The schema (`05:161-380`) is genuinely comprehensive and well-designed for the domain: separate inpatient per-diem / case-rate / other buckets, outpatient surgical tiers + ER + infusion + therapy + radiology formula components, capitation by aid category with multi-year escalation, stop-loss, exception provisions (implants/drugs), HAC/never-events, chargemaster provisions, and verbatim `full_clause_text` for the six legally-sensitive clause families. The four document-type-specific prompts (base / amendment / cover-memo / settlement) are appropriately specialized, and `temperature=0.0` (`05:431`) is the right call.

The problem is the **defaulting instructions**:

- *"Use 'Commercial' if not specified"* for program and *"All Networks" if not specified* for network (`04:33-34`). The model is **instructed to fabricate** a program and network when the contract is silent. This is hallucination-by-design, and it is then **reinforced at the table layer** (`08:485-493`) where NULL/empty program is overwritten with `'Commercial'` and network with `'All Networks'`.
- *"Use the contract or exhibit effective date if no rate-specific date"* (`04:32`) conflates a rate's economic effective date with the document's date.

The downstream code itself testifies to how unreliable these become: the V8 dedup **deliberately removes program and network from its dedup key** because "LLM assigns inconsistently across extractions" (`12:172-176`). The pipeline fabricates these fields, then has to ignore them.

JSON robustness is handled by `_repair_truncated_json` (`05:452-486`), which closes open brackets when output hits `max_tokens` (65,536). That keeps the file parseable but **silently truncates trailing rate rows** — a large contract whose output overflows loses its last rates with no error surfaced, only a `json_repaired` flag.

---

## 2. Clause and reimbursement modeling

### Clause extraction — strong on preservation, weak on structure
The prompts force **verbatim full-clause text** for termination, compliance, grievance/appeals, dispute resolution, confidentiality, and indemnification (`04:37-45`). This is the strongest single design decision in the system: it preserves the actual legal language rather than a lossy summary, which is exactly right for a legal QA / retrieval use case. In `09_build_terms` these become rows in `tbl_contract_clauses_terms`, later unified into `tbl_genie_contract_terms`.

The weakness is that clauses are modeled as **flat text rows keyed by topic**, with no clause-level effective dating, no cross-reference resolution (definitions referenced inside a clause aren't linked), and no amendment linkage at the clause level. A termination clause amended in year 3 produces a *new row from a newer document*; "which termination language is in force today" is answered only by the document-level `is_from_latest_doc` flag (12), not by any clause-level supersession.

### Reimbursement extraction — rich schema, lossy flattening
`make_rate_row` (`08:158-217`) flattens the nested rate structures into one wide `tbl_contract_rates_all`. Two real correctness issues live here:

- **Falsy-value bug in the value-selection chains.** `rate_numeric_val = item.get('rate_numeric') or item.get('stop_loss_numeric') or item.get('cap_numeric') or ...` (`08:180-184`). Because `0` is falsy, a legitimate `$0` rate (or a `0.0`) falls through and grabs the *next* field — e.g. a stop-loss or cap value — silently mislabeling the row. The text chain (`08:170-176`) has the same shape.
- **Compound rates collapse to one value.** Emergency services are typically "X% of charges, capped at $Y." The chain takes `percentage_of_charges` *or* `cap_amount`, not both (`08:173-176`), so one half of the ER reimbursement rule is dropped. Radiology "conversion factor × multiplier" is partly rescued into a `formula` string (`08:186-193`) but is no longer computable.

`program_normalized` (`08:521-541`) maps 148 raw variants to 8 buckets via ordered `LIKE` rules. There is an **ordering bug**: the `Medi-Cal` rule (`%medi-cal%`, line 524) precedes the `Dual Eligible` rule (`%medicare% AND %medi-cal%`, line 525), so a "Medicare and Medi-Cal" program matches Medi-Cal first and **never reaches Dual Eligible**. Combined with the Commercial-defaulting, the program dimension is systematically skewed toward Commercial and away from Dual Eligible.

### Provider normalization (10) — overstated
The markdown claims names are normalized by "strip legal suffixes, whitespace, case" (`10:13`). **The SQL does none of that.** Identity resolution picks each id's latest-document name (`10:205-216`) and then groups by **exact `canonical_name` string equality** (`10:218-226`). So "Sutter Health", "Sutter Health, Inc.", and "SUTTER HEALTH" become three different facilities, while two genuinely distinct facilities sharing a generic extracted name could be merged. The 456→278 resolution is real but is exact-string matching on filename-derived names — brittle, and weaker than the documentation implies.

---

## 3. Amendment intelligence — the core evaluation

This is where "enterprise-grade vs prototype-grade" is decided. There are **two different supersession mechanisms** in the codebase, they are inconsistent, and the more sophisticated one covers only part of the data.

### Amendment ordering is lexical, from filenames (07:127-149)
`get_amendment_order` maps ordinal **words in the filename-derived doc_type** to integers ("first"→1 … "twentieth"→20), base→0, settlement→900, letter/mutual→800, **everything unrecognized→500** (`07:149`). Consequences:

- Ordering is **not chronological** — it's lexical. An amendment whose filename lacks a recognized ordinal lands at 500 alongside every other unrecognized doc, so their relative order is undefined.
- A cover memo "FirstAmendCM" and the actual "FirstAmend" can both resolve to order 1 → ties.
- 11 then treats orders ≥100 as **sentinels excluded from `amendment_depth`** (`11:119`), confirming that a meaningful share of documents have no usable position in the chain.

### Supersession Mechanism A — key-based, in consolidation (07:281-307)
This is the "real" engine. Documents are sorted (base-first, then `amendment_order`, then parsed effective date; `07:270-274`), and as rates are walked in order, a later rate **supersedes an earlier one only when they share the exact tuple** `(rate_type, service_category, program, network)` (`07:288`). Three structural problems:

1. **`service_category` is free-text from the LLM.** "Med/Surg", "Medical/Surgical", "Medical/Surgical/Pediatrics" are different keys, so a restated rate often **fails to supersede** its predecessor and both remain `current` — this is the origin of the ~27.6% duplicate-current problem the V8 dedup later tries to mop up.
2. **`program`/`network` are in the key but are unreliable/empty at this stage**, so supersession matching is inconsistent (matches when both empty, misses when one got defaulted).
3. **Coverage is partial.** `extract_rates_summary` (`07:183-229`) only emits four categories: inpatient per-diem, inpatient case-rate, outpatient surgical tiers, and capitation `rates_by_category`. ER, infusion, therapy, radiology, other-inpatient/outpatient, and multi-year capitation are **never seen by Mechanism A**.

### Supersession Mechanism B — document-level, in build_rates (08:452-482)
Every rate *not* carrying a status from consolidation (i.e. all the categories Mechanism A ignores) gets status filled by a blunt rule: **if the rate's document is the provider's max `amendment_order`, it's `current`; otherwise `superseded`** (`08:472-482`). This hard-codes the assumption that **every amendment is a full restatement of the entire fee schedule**. When an amendment modifies only one exhibit (e.g. just the capitation table), this rule marks *all* base-agreement inpatient per-diems `superseded` even though they were never replaced — and because the provider still has *some* current rate, the zero-current safety net (`08:576-604`) does **not** fire to correct it. The result is silent loss of in-force rates from current-state.

### Consolidation runs per `provider_id`, not per facility (07:169-172)
Amendment chains are built per id. Identity resolution (10) runs *afterward* and never feeds back. So for a facility split across multiple provider_ids, an amendment filed under one id **cannot supersede** rates under another id; the temporal chain is fragmented across the very ids that `dim_provider_canonical` exists to merge.

### `amendments_impact` is captured but inert
The amendment prompt carefully elicits `exhibits_replaced`, `sections_modified`, `prior_rates_superseded`, and `supersedes_effective_date` (`04:115-122`), and 10 stores them on `documents_master`; 11 even parses `supersedes_effective_date_parsed` (`11:357-364`). **None of it drives supersession.** The system extracts what each amendment says it changes and then ignores that signal when computing what is actually in force. This is the largest gap between conceptual design and implemented behavior.

### Current-state reconstruction (12) is a downstream patch
`tbl_genie_rates_current` re-derives "current" by filtering `status='current'` and de-duplicating with a window partitioned by `(facility, rate_category, service_category, rate_numeric|rate_text, effective_date_parsed)`, keeping the latest amendment (`12:159-205`). It joins through `dim_provider_canonical`, so dedup *does* operate at facility level — partially compensating for the per-id consolidation gap, but only for dedup, never re-running supersession. Note `effective_date_parsed` is **in the partition key**, which undercuts the stated goal of collapsing the same dollar rate restated across amendments (each restatement carries its own date → separate partitions → both survive), which is exactly why ~3.7% residual duplicates remain.

---

## 4. Implementation status

| Capability | Status | Evidence |
|---|---|---|
| 2-tier OCR with checkpointing | **Implemented** | `02` full |
| Verbatim clause preservation | **Implemented** | `04:37-45`, schema `05` |
| Comprehensive rate schema | **Implemented** | `05:161-380` |
| Per-PDF LLM extraction + retry + JSON repair | **Implemented** | `05:489-612` |
| Per-provider consolidation + rate timeline | **Implemented** | `07` |
| Base/Genie/serving Delta build | **Implemented** | `08-13` |
| Provider identity resolution | **Partial** — exact-string only, not the documented name-normalization | `10:205-226` vs `10:13` |
| Amendment chain ordering | **Partial** — lexical from filenames; large sentinel bucket | `07:127-149`, `11:119` |
| Rate supersession | **Partial & inconsistent** — two mechanisms, one partial-coverage and key-fragile, one full-restatement assumption | `07:281-307`, `08:452-482` |
| Current-state reconstruction | **Partial** — downstream dedup, can't recover partial-amendment in-force rates | `08:472-482`, `12:159-205` |
| Exhibit/section-level supersession (`amendments_impact`) | **Conceptual only** — extracted, stored, never used | `04:115-122`, `10:134-141`, unused |
| Compound/formula reimbursement (ER cap+%, radiology) | **Conceptual/partial** — collapsed to one value or a non-computable string | `08:170-193` |
| Content-based document-type classification | **Missing** — 100% filename-derived | `03`, `04:243-280` |
| Clause-level temporal state | **Missing** — document-level `is_from_latest_doc` only | `12` |

---

## 5. Logical gaps, brittle parsing, hallucination risks

**Brittle parsing.** (1) Filename regex is the spine for type/order/identity (`03:38`) — any naming drift cascades. (2) Lexical amendment ordering with a 500 catch-all (`07:149`). (3) Exact-string facility grouping (`10:218`). (4) Free-text `service_category` as a supersession key (`07:288`). (5) Layout-blind table flattening in OCR (`02:160`).

**Logical gaps.** (1) `amendments_impact` never wired to supersession. (2) Consolidation per-id, not per-facility. (3) Full-restatement assumption breaks partial amendments with no detection. (4) Two inconsistent supersession regimes in one table. (5) Chunk-merge drops scalar corrections and duplicates list rows.

**Hallucination risks.** (1) Program→Commercial / network→All Networks fabrication, at both prompt and table layers (`04:33-34`, `08:485-493`) — these are *invented* facts presented as extracted. (2) Per-chunk full-schema extraction invites each chunk to emit sections it only partially saw (`05:576-582`). (3) `_repair_truncated_json` masks silent rate loss as success (`05:452-486`). (4) The `0`-is-falsy value chains can attach the wrong dollar figure to a row (`08:180-184`). Mitigants present: `temperature=0`, "extract only what is written" date/name/payment rules (`04:79-81`), and verbatim-text requirements all push the other way and are well done.

---

## 6. Does the model represent the true legal/business semantics of provider contracts?

**Partially — it represents the *content* of contracts well and the *temporal legal state* of contracts poorly.**

**Legal meaning being lost.** A provider agreement is a *base contract modified by an ordered chain of amendments, each of which surgically replaces specific exhibits/sections effective on its own date*. The model flattens this to "rows tagged by the document they came from, with the latest document's rows called current." Specifically lost: (a) **which amendment changed what** — the `amendments_impact` signal is inert, so the system cannot answer "what did the 3rd amendment actually do"; (b) **partial amendments** — a contract where most terms survive untouched is misrepresented as wholesale superseded; (c) **clause-level currency** — only document-level latest-ness exists; (d) **defined-term scope** — definitions are stored but not linked to the clauses that rely on them, so the operative meaning of a clause can change without detection.

**Reimbursement semantics insufficiently modeled.** Reimbursement is conditional and compound: "X% of billed charges up to a $Y cap, subject to a stop-loss attachment at $Z, with carve-outs for implants above $W." The schema has the *fields* for these but the flattening (`make_rate_row`) reduces a rate to a single `rate_text`/`rate_numeric`, dropping the conditional structure (cap+% collapse, falsy-zero bug). Fabricated program/network then attach false applicability scope to rates. The net effect: individual dollar figures are often captured, but the **rule that turns a claim into a payment** is not faithfully modeled.

**Operational semantics missing.** DOFR (division of financial responsibility), risk-sharing/withhold mechanics, timely-filing and clean-claim windows, chargemaster-change governance, and delegated activities are captured as flat text fields on `documents_master` but are not modeled as enforceable, queryable rules, and carry no temporal state. They exist as searchable strings, not as operational logic.

**Is temporal legal state truly reconstructable?** **No — not reliably.** To reconstruct "what was in force on date D for facility F" you need: facility-level identity (only partially correct), chronological amendment ordering (lexical/filename, with sentinels), exhibit/section-level supersession (not implemented), and clause-level effective dating (not implemented). The system can answer "show me the latest document's rates" and "show me rate history," but it cannot reliably answer "what reimbursement rule governed this admission on this date," which is the actual enterprise question.

**Enterprise-grade or prototype-grade?** **Prototype-grade for amendment intelligence; near-production-grade for extraction-and-retrieval.** The ingestion, schema, clause preservation, and table-building are solid and scalable. The amendment/temporal layer rests on filename parsing, a free-text supersession key, a full-restatement assumption, and an unused amendment-impact signal — the hallmarks of a capable demo, not a system of record for "what is contractually in force."

---

## 7. Strongest parts of the current model

The **verbatim clause preservation** (`04:37-45`) is the right legal-AI instinct and the foundation that makes grounded QA possible. The **comprehensive, domain-aware JSON schema** (`05:161-380`) captures the real structure of California provider agreements (per-diem/case/capitation/outpatient tiers/stop-loss/HAC/chargemaster/DOFR). The **engineering discipline** — checkpointing, adaptive concurrency, per-file timeouts, JSON repair, idempotent overwrites, `temperature=0`, explicit "extract only what is written" guards — is genuinely production-minded. And the **layered table model** (base → genie → serving) with a canonical provider dimension is a clean, queryable foundation. The honest self-correction in the V8 dedup (dropping unreliable fields from the key) shows the authors understand where their own extraction is weak.

## 8. The biggest semantic weaknesses

In priority order: (1) **Amendment-impact is extracted but never drives supersession** — the system ignores what amendments say they change. (2) **Document-level full-restatement supersession** silently drops in-force rates from partial amendments. (3) **Free-text `service_category` as a supersession/dedup key** produces both false duplicates and false supersessions. (4) **Fabricated program/network** inject false applicability and pollute the program dimension. (5) **Consolidation per provider_id, not per facility**, fragments the chain. (6) **Filename-derived everything** (type, order, identity) with no content-based validation. (7) **Compound reimbursement rules collapsed** to single values.

## 9. How the semantic model should evolve

The unifying idea: **stop modeling contracts as a pile of document-tagged rows and start modeling them as a bitemporal, exhibit-aware contract state machine.**

1. **Make `amendments_impact` authoritative.** Drive supersession from each amendment's declared `exhibits_replaced` / `sections_modified` / `supersedes_effective_date`, not from "is this the latest document." This directly fixes the partial-amendment failure and unlocks "what did amendment N change."
2. **Introduce a stable rate identity (a service key) instead of free-text `service_category`.** Normalize service lines + revenue codes into a controlled vocabulary so restatement detection is exact, and supersession/dedup stop fighting LLM text variance.
3. **Model temporal state bitemporally** — every rate and clause carries `effective_from` / `effective_to` (and ideally knowledge dates) so "in force on date D" is a direct query rather than a "latest document" approximation.
4. **Resolve identity before consolidation, at facility level, with fuzzy/normalized matching** (suffix stripping, tax-id/NPI anchoring) so amendment chains are built per facility, not per id.
5. **Add a content-based document classifier** to validate (not replace) the filename type, and to recover the large sentinel-order population into a real chronology.
6. **Preserve compound reimbursement structure** — model "% of charges + cap", stop-loss attachment, and formula rates as structured conditional objects, not a single number, and remove the program/network defaulting (carry explicit "unspecified" instead of fabricating "Commercial").
7. **Push supersession down to the clause and exhibit level**, and link defined terms to the clauses that use them, so legal currency is answerable at the granularity lawyers actually work in.
8. **Replace length-based OCR reconciliation and table flattening** with layout/table-aware parsing so rate grids survive ingestion as tables.

Steps 1–3 are the highest-leverage: they convert the amendment layer from prototype to enterprise-grade and make true temporal legal-state reconstruction possible.

---

*Method note: findings are grounded in direct reading of notebooks 02, 03, 04, 05, 07, 08, 10, 11, and 12 (with 16's chunker). Row counts and percentages referenced are the repo's own reported figures, not independently re-derived against a live catalog.*
