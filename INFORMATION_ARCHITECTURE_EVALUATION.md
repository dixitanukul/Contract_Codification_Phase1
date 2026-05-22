# Information Architecture Evaluation & Target Architecture (Phase 3)

*Critical evaluation of whether the platform represents provider-contract intelligence in the best possible way, plus a proposed enterprise target architecture. Grounded in the repository implementation (notebooks 02–25), the corpus summary, and real contract structure. Citations are `notebook:line` or "corpus" for `CONTRACT_CORPUS_SUMMARY.md`.*

---

## The central thesis

The platform is a **well-built retrieval system sitting on top of a flattened analytical warehouse**. It is organized around the question *"find me text/rows similar to this query"* when enterprise legal intelligence requires a system organized around *"what is the governing contractual state for facility F, program P, service S, on date D — and prove it from the source."* That is the difference between a **retrieval-centric** architecture and a **state-centric, graph-aware** one. The current design is the former; the domain demands the latter.

Two corpus facts make this decisive. First, the structured model is built from only **3,519 of 10,372 files (34%)** (corpus §12) — the warehouse represents a third of the contracts and silently treats that third as the world. Second, the **documents themselves declare their supersession in language** — "replaces in entirety" (59.5%), "supersedes" (48.5%), "in lieu of" (43.6%), "hereby amended" (30.2%) (corpus §8) — and the architecture captures *none* of that declared structure as state; it relies instead on "row came from the latest-numbered document."

---

## Part A — The eleven evaluation questions

**1. Is the Delta table design semantically correct?**
Partially. The medallion-style layering (base → genie → serving) is a correct *engineering* pattern, and `dim_provider_canonical` is the right instinct. But the schema is semantically wrong in one fundamental way: **the contract is not an entity in the model.** The corpus has 4,224 distinct contracts across 531 provider IDs (corpus §1, §8), yet there is no `contract` table — the spine is `tbl_contract_documents_master` (one row per *PDF*) and rates/clauses are keyed to `provider_id` + `source_filename`. Consolidation even groups amendment chains by `provider_id` (`07:169-172`), so a provider's *multiple distinct contracts* are merged into one timeline and their rates can supersede each other across contract boundaries. A model of contracts that has no contract entity is semantically incorrect at the root.

**2. Is the representation too flat?**
Yes, severely. The terminal artifact `tbl_genie_contract_terms` collapses clauses, medical codes, HAC events, and parties into a single table whose `content_text` is a `CONCAT(...)` string (`12:243-330`). A party's tax-id/NPI/signatory facts, a HAC adjustment rule, and a verbatim termination clause are all reduced to one untyped text column distinguished only by a `content_type` tag. Rates are one wide row in `tbl_contract_rates_all` with `rate_text`/`rate_numeric` (`08:195-217`). The richly nested extraction JSON — which *did* preserve structure — is flattened on the way into Delta and never recovered. The vector index is then built on top of that flattened concat (`16:86`, `STRUCTURED_SOURCE = tbl_genie_contract_terms`), so the loss propagates into retrieval.

**3. Is important legal structure being lost?**
Yes. Real BSC agreements are hierarchical: *Agreement → numbered Articles (I. DEFINITIONS … X. TERM & TERMINATION) → Exhibits (A, C, G) → rate schedules* (corpus §4). Exhibit references appear in 88% of files and schedules in 69.5%, while inline rate tables appear in only 19.6% (corpus §6) — meaning **most rate truth lives in referenced exhibits, and the exhibit is the unit amendments replace.** The model has `exhibit_reference` only as a free-text string on a rate row; there is no exhibit entity, no section entity, no definition-to-clause linkage. The document graph that lawyers actually navigate is discarded into flat rows.

**4. Is reimbursement intelligence modeled correctly?**
No — it is captured but not *modeled*. Reimbursement is conditional and compound ("X% of charges up to $Y cap, with stop-loss attachment at $Z, carve-outs for implants > $W"), and a single contract typically mixes FFS, per-diem, case, DRG, and capitation (corpus §5). The schema has fields for these but the flattening reduces each to one scalar (the ER cap-vs-percentage collapse and falsy-`0` bug at `08:170-193`). **DRG appears in 42.8% of files (corpus §5) yet has no first-class rate structure** in the schema (`05:190-232`). Program/network are fabricated by default (`04:33-34`, `08:485-493`) then dropped from dedup as unreliable (`12:172-176`). The model can tell you *a number that appeared*; it cannot reliably tell you *the rule that adjudicates a claim*.

**5. Is amendment state modeled correctly?**
No — this is the weakest area (detailed in the Phase 2 audit). Ordering is lexical from filenames (`07:127-149`), supersession runs in two inconsistent mechanisms (key-based on free-text `service_category` in `07:288`; document-level full-restatement in `08:472-482`), and the amendments' own declared impact (`exhibits_replaced`, `sections_modified`, `supersedes_effective_date`) is extracted and stored but **never used** (`04:115-122` → `10:134-141`, inert). The corpus shows 90% of amendments contain explicit supersession language (corpus §8) — the signal is right there in the text and the model ignores it.

**6. Is the system truly reconstructing CURRENT contract state?**
No. "Current" is operationalized as *"the row came from the provider's highest-amendment-order document"* (`08:472-482`, `12:159-205`). This is an approximation that fails on the common case of a **partial amendment**: when an amendment changes only one exhibit, the still-in-force base rates get marked superseded and the zero-current safety net doesn't fire (`08:576-604`). There is no point-in-time query ("state as of date D"); there is only "latest document's rows." True current-state reconstruction would require exhibit-level supersession applied along a chronological chain — neither exists.

**7. Are clauses represented at the correct abstraction level?**
No. Clauses are document-level text rows with a `topic` tag and an `is_from_latest_doc` boolean computed per `provider_id` (`12:246-256`). The correct abstraction is a **versioned clause bound to a contract section, with effective dating and amendment lineage**. As built, you cannot ask "what is the operative termination provision for this contract today and which amendment last changed it" — you can only retrieve text rows and read the latest-doc flag. Defined terms (`key_definitions`) are stored but not linked to the clauses that depend on them, so a clause's *meaning* can shift silently.

**8. Are provider relationships modeled properly?**
Partially, and weaker than documented. `dim_provider_canonical` resolves IDs→facilities by **exact latest-name string match** (`10:218-226`), not the suffix/case normalization the markdown claims (`10:13`), and not via Tax-ID/NPI anchoring even though those exist in the data (corpus §6: Tax IDs in 20.8%, NPI in 44.5%). Real provider structure is a hierarchy — *health system → facility → contracting entity (Tax ID/NPI) → provider_id* — and includes the IPA-vs-direct distinction that the corpus calls "fundamentally different" (corpus §9: IPA is 80% capitation with withholds/risk-sharing). None of that relational structure is a first-class dimension.

**9. Are operational obligations modeled properly?**
No. Timely-filing windows, clean-claim periods, chargemaster-change governance, DOFR (division of financial responsibility), delegated activities, withhold/risk-sharing mechanics, and notice periods are captured as flat text columns on `documents_master` (`10:113-172`). They are searchable strings, not queryable, dated, enforceable rules. "Which facilities owe us encounter data within 30 days and are currently out of compliance" is unanswerable from this model.

**10. Is the architecture retrieval-centric when it should be state-centric/graph-centric?**
Yes — this is the core architectural mismatch. The terminal consumables are two vector indexes (`16`) plus flat SQL tables, and the flagship app (23, grounded QA) is RAG: expand query → `similarity_search` over embedded flattened text → LLM synthesis (`23:78-223`). Grounding is to *retrieved text*, not to a *contractual state*. The right center of gravity is a **bitemporal contract state store + a contract knowledge graph**, with retrieval as one access path into it — not the organizing principle.

**11. Is the current model sufficient for enterprise legal intelligence?**
Not yet. It is sufficient for *contract search and summarization over the extracted subset*. It is insufficient as a **system of record for what is contractually in force**, because it cannot reconstruct point-in-time state, cannot trace supersession at exhibit/clause granularity, has no contract entity, fabricates applicability fields, and covers only 34% of the corpus. For negotiation, renewal, audit defense, or reimbursement adjudication — the stated goals — that is a prototype, not an enterprise system.

---

## Part B — What it gets right, and what it fundamentally misses

**Gets right.** Verbatim clause preservation (`04:37-45`) — the single best decision, and the foundation any better architecture would keep. A genuinely domain-aware extraction schema (`05:161-380`). Production-minded engineering (checkpointing, adaptive concurrency, `temperature=0`, idempotent rebuilds, JSON repair). A sensible layered warehouse with a canonical provider dimension. Honest self-awareness about its own weak fields (dropping program/network from the dedup key). A working hybrid retrieval + Genie surface that demos well.

**Fundamentally misses.** (1) **The contract as a first-class, versioned entity** — everything is keyed to PDFs and provider IDs. (2) **Declared supersession** — the corpus's explicit "replaces/supersedes/in lieu of" language and the extracted `amendments_impact` are both unused. (3) **Point-in-time state** — there is no bitemporal model, so "as of date D" cannot be answered. (4) **The document/exhibit/section/clause graph** — flattened away. (5) **Reimbursement as adjudicable rules** — collapsed to scalars. (6) **Coverage** — 66% of the corpus is absent and treated as nonexistent.

---

## Part C — Proposed superior architecture

The unifying move: **shift the center of gravity from an index to a bitemporal contract knowledge graph, and make retrieval, analytics, and QA all views onto that single source of truth.**

```
                      ┌──────────────────────────────────────────────────────┐
   SOURCE             │  PDFs + exhibits (10,372 files, 4,224 contracts)       │
                      └──────────────────────────────────────────────────────┘
                                          │ layout/table-aware OCR
   EXTRACTION         ┌──────────────────────────────────────────────────────┐
   (evidence layer)   │  Document AST: doc → article → section → exhibit →     │
                      │  clause / rate-schedule / definition (with spans+page) │
                      └──────────────────────────────────────────────────────┘
                                          │ classify · resolve · link
   CANONICAL GRAPH    ┌──────────────────────────────────────────────────────┐
   (state layer)      │  KNOWLEDGE GRAPH + BITEMPORAL STATE                    │
                      │  HealthSystem→Facility→ContractingEntity(TIN/NPI)      │
                      │       →Contract→{Exhibit, Section, Clause, RateRule}   │
                      │  every node: effective_from/to + knowledge_from/to     │
                      │  edges: AMENDS, REPLACES_EXHIBIT, SUPERSEDES,          │
                      │         DEFINES, REFERENCES, DELEGATES                 │
                      └──────────────────────────────────────────────────────┘
                            │                    │                    │
   SERVING        point-in-time state API   analytical marts    embeddings/graph
                  "state(F,P,S,date)"       (rates, ops, risk)   index (clause/rule
                            │                    │                nodes w/ state tags)
   INTELLIGENCE   ┌──────────────────────────────────────────────────────────┐
                  │  State-grounded QA · negotiation · renewal · audit         │
                  │  every answer = state node(s) + provenance span + as-of    │
                  └──────────────────────────────────────────────────────────┘
```

**1. Superior enterprise semantic architecture.** Keep the medallion lake for evidence (bronze = OCR/AST, silver = normalized facts), but make the **gold layer a bitemporal knowledge graph**, not flat marts. Marts and vector indexes are *projections* of the graph, regenerated from it — so there is exactly one definition of "current" and it lives in one place. This eliminates the current situation where "current" is computed three different ways (07, 08, 12).

**2. Superior contract representation model.** Promote the hierarchy the documents already have into entities: `Contract` (the durable agreement, keyed by `contract_id`) owns `Document` versions; each `Document` contributes `Section`/`Exhibit`/`Clause`/`RateRule` nodes carrying their text span and page provenance. A clause is a *node* with versions, not a text row. Definitions link to the clauses that use them (`DEFINES` edges). This restores question 3's lost structure and fixes question 1's missing contract entity.

**3. Superior amendment model — event-sourced, declaration-driven.** Treat each amendment as an **event** that asserts typed operations parsed from its own language and `amendments_impact`: `REPLACE_EXHIBIT(C)`, `MODIFY_SECTION(5.2)`, `RESTATE_ALL`, `SET_RATE(...)`, effective on its stated date. Current state is the **fold of these events over the chain**, per contract, at exhibit/clause granularity. This makes partial amendments correct (only the replaced exhibit's rates supersede), uses the 90% of amendments that declare supersession (corpus §8), and makes "what did amendment N change" and "state as of date D" first-class queries. Free-text `service_category` is replaced by a stable service key so restatement matching is exact.

**4. Superior reimbursement intelligence model.** Replace the single wide rate row with a **structured payment-rule object**: `{method (FFS|per_diem|case|DRG|capitation|%charges|%Medicare), service_key, conditions (caps, stop-loss attachment, carve-outs), program, network, exhibit_version, effective interval, provenance}`. Model DRG explicitly (43% of corpus). Carry an explicit `unspecified` for program/network instead of fabricating "Commercial." Separate IPA (capitation/withhold/risk) and direct (FFS/per-diem/DRG) archetypes as a first-class contract attribute (corpus §9). The goal: a rule object precise enough to *adjudicate a sample claim*, which is the real test of reimbursement intelligence.

**5. Superior retrieval abstraction — state-aware hybrid.** Index **state nodes**, not flattened document text: each embedded chunk is a clause/rule node carrying its `contract_id`, `effective interval`, `is_in_force` flag, and provenance span. Retrieval filters by entity + as-of date *before* semantic ranking, so "termination clause for Sutter in force today" returns the governing node, not every historical variant. Graph traversal (definitions, referenced exhibits, prior versions) augments vector recall. Retrieval becomes an access path into the state layer rather than the product.

**6. Superior serving abstraction — a point-in-time state service.** The primary serving contract is a function: `contract_state(facility, program, service, as_of_date) → {rate_rule, governing_clause, source_amendment, provenance}`. Dashboards and Genie call this, so every surface shares one notion of truth. The flat serving tables become cached projections with an `as_of` column, not hand-built dedup tables with bespoke "current" logic.

**7. Superior user-facing intelligence abstraction.** Move from "chat that retrieves text" to **state-grounded answers**: every response resolves to specific state node(s), shows the *as-of date*, names the *governing amendment*, and links the *verbatim provenance span*. Confidence and coverage are surfaced ("based on 4 of 6 amendments extracted; exhibit C unresolved"). This converts the system from a search assistant into an auditable legal-intelligence tool a negotiator or auditor can rely on and defend.

---

## Part D — What enterprise legal AI systems would do differently

Mature contract-intelligence platforms (CLM/legal-AI) converge on a few patterns this system lacks. They treat the **contract, not the document, as the aggregate root**, with documents as versioned events against it. They maintain an **obligation/clause model with effective dating and provenance** rather than free text, so "what's in force" is a query, not a heuristic. They use **layout- and table-aware extraction** so rate grids survive as structured tables. They anchor **entity resolution on hard identifiers** (Tax ID/NPI/registry IDs) with fuzzy names as a secondary signal. They keep a **human-in-the-loop review and confidence/coverage ledger** because legal state must be auditable. They are **bitemporal** (effective time + knowledge time) so they can answer "what did we believe was in force, as of when." And they treat **retrieval/RAG as one interface onto a governed knowledge layer**, never as the system of record. The current platform has the extraction and retrieval pieces but is missing the governed state layer those systems are built around.

---

## Part E — What should evolve next (sequencing)

1. **Introduce the `Contract` entity and re-key consolidation to `contract_id`** (fixes cross-contract supersession bleed; cheap, high impact).
2. **Make `amendments_impact` authoritative** and fold amendment events at exhibit level (fixes partial-amendment current-state).
3. **Add bitemporal effective intervals** to rates and clauses → enables point-in-time queries.
4. **Replace free-text `service_category` with a controlled service key**; model DRG and compound reimbursement as rule objects; stop fabricating program/network.
5. **Stand up the point-in-time state API** and regenerate serving tables + vector index as projections of it.
6. **Re-anchor identity on Tax ID/NPI** and add the system→facility→entity hierarchy and IPA/direct archetype.
7. **Close the coverage gap** (extract the remaining 66%) and add a **confidence/coverage ledger** to every answer.
8. **Upgrade ingestion to table-aware parsing** so exhibits/rate grids survive.

Steps 1–3 are the inflection point: they convert the platform from a retrieval system over a flat warehouse into a state-centric system of record — the prerequisite for genuine negotiation, renewal, audit, and reimbursement intelligence.

---

*Method note: grounded in notebooks 02–25 and `CONTRACT_CORPUS_SUMMARY.md`. Corpus counts (10,372 files, 4,224 contracts, 34% extracted, supersession-language frequencies, payment-method mix) are the repo's own reported figures, used here to assess fit between the data's real structure and the model's representation.*
