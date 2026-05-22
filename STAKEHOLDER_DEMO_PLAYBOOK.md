# Stakeholder Demo Playbook (Phase 7)

*How to demo the Provider Contract Intelligence Platform next week so it lands as enterprise-grade — showing the genuinely strong capabilities, steering around the brittle ones, and handling questions with accurate scoping. Builds on the six prior phase audits. Function/notebook references are for the presenter.*

---

## The one-sentence framing

> "We've turned ~10,000 unstructured provider contracts into a queryable, semantically searchable contract intelligence layer — with the actual legal language preserved and citable — and built the first wave of negotiation, renewal, and clause-analysis tools on top of it."

That sentence is *true today* and is the safe envelope for the whole demo. Everything below keeps you inside it.

---

## 1. What is genuinely impressive already

- **Scale of ingestion.** ~10,349 PDFs (29 GB, 425K pages) OCR'd at 100% success via a real two-tier pipeline with checkpointing and adaptive concurrency (corpus §1, §11). This is hard, real data engineering — lead with the scale.
- **Verbatim legal-language preservation.** The pipeline keeps the exact contract text (not summaries) for termination, dispute, confidentiality, compliance, indemnification, etc. (`04:37-45`). This is the foundation of trustworthy legal AI and most pipelines don't do it.
- **A genuinely domain-aware data model.** Rates split into per-diem / case / capitation / outpatient-tier / stop-loss / HAC / chargemaster / DOFR, normalized across hundreds of providers into clean Delta tables with a canonical provider dimension (08–13). Having this substrate at all is the hard part of contract intelligence.
- **Managed semantic search.** Two Databricks Vector Search indexes on `gte-large-en` over labeled clauses and full OCR text (16) — a production-style retrieval substrate, not a toy FAISS index.
- **Breadth of attempted intelligence.** Clause search, clause-standardization/outlier detection, citation QA, amendment timelines, comparison, renewal, and negotiation are all attempted on one foundation — the right surface area for an enterprise pitch.

## 2. Workflows that should absolutely be demoed

1. **Clause standardization / outlier detection (22).** "Is our termination language standard or unusual across the network?" → ranks providers from most-standard to most-unusual. Visually compelling, semantically real, low failure risk. **This is your hero demo.**
2. **Semantic clause search (17).** "Find arbitration / dispute-resolution language across all providers" → ranked clause hits with source documents. Shows the embedding layer doing real work.
3. **Citation-style clause QA (20), scoped to language.** "What do these contracts say about confidentiality obligations?" → grounded answer with inline `[n]` citations to source PDFs. Impressive *when framed as a language question*.
4. **Amendment timeline view (21, `get_amendment_timeline`).** For a well-amended example provider, show the full base→amendment chain with dates and document types. Tells the "we reconstructed contract history" story cleanly.
5. **Descriptive provider/rate lookup (Genie tables).** "Show current rate categories and stop-loss for Garfield" — straightforward, reliable SQL over the example providers.

## 3. What is risky to demo

- **"What rate is in force *today* for X service at provider Y?"** — current-state is a heuristic; partial-amendment services can be wrongly superseded or wrongly current.
- **Historical amendment *diff* (21's rate diff).** It reads the current-only table, so comparing two past amendments is structurally broken — shows everything as "added." Demo the *timeline*, not the rate diff.
- **Rate trends / escalation over time (24).** No real time-series exists; "escalation" is a level-vs-network number.
- **Dollar-impact / savings claims (25 proposals).** No claims/utilization volume in the platform — there is no defensible dollar figure.
- **Anything implying full coverage.** Only ~34% of the corpus is extracted; "we cover 278 providers" reads as complete but is the extracted subset.
- **Citation tool on a superseded provision** — it can label historical text "CURRENT" (`20:189` hardcode).
- **Rate questions through the conversational QA SQL path (23)** — errors on a non-existent column and silently falls back to text.
- **Live queries against arbitrary/obscure providers** — stay on curated examples (Garfield, Kaweah Delta, St Josephs, UC Davis, Grossmont).

## 4. What creates the strongest perception of enterprise intelligence

- **Provenance.** Every answer tracing back to a named source PDF and page makes it feel auditable and trustworthy — far more than a slick answer alone. Lean hard on citations.
- **Scale numbers.** "10,349 contracts, 425,000 pages, 597 providers" establishes this as an enterprise corpus, not a sandbox.
- **The network view.** "Across *all* providers, here's who is the outlier on termination language" — cross-portfolio comparison is something no individual lawyer can do manually and instantly signals leverage.
- **Domain specificity.** Showing the system understands per-diems, stop-loss, capitation, DOFR, HAC — payer-contracting vocabulary — signals this was built for *their* problem, not a generic demo.

## 5. What differentiates this from a generic RAG demo

Generic RAG = "upload PDFs, chat with them." Your differentiation, all demonstrable:

- **A structured contract data model**, not just an index — you can run analytics (benchmarks, comparisons, outliers), not only Q&A.
- **Verbatim legal-clause preservation** with labeled clause types — retrieval over *legal units*, not arbitrary text blobs.
- **Cross-portfolio intelligence** — "standard vs unusual across 278 providers" is a fleet-level capability generic RAG can't do.
- **Amendment-aware framing** — you reconstruct contract *history*, not just current text.
- **Domain depth** — the schema and vocabulary are healthcare-payer-specific.

Say it plainly: *"This isn't chat-with-your-PDFs. It's a structured contract intelligence layer that also answers questions."*

## 6. The story to tell stakeholders

A three-act arc:

1. **The problem.** "Blue Shield holds thousands of provider contracts as scanned PDFs across hundreds of folders. Today, answering 'what does our network look like on termination terms' or 'where do this provider's rates sit' means someone reading PDFs for days."
2. **What we built.** "We ingested the entire corpus, preserved the exact legal language, structured it into a queryable model, and made it semantically searchable — so the contracts become an asset you can interrogate."
3. **What it unlocks.** "On top of that foundation we've built the first intelligence tools — clause standardization, semantic search, grounded Q&A, amendment timelines — and there's a clear roadmap to negotiation and renewal intelligence as we deepen the model." (Position negotiation/renewal as *direction*, not finished product — that's both honest and a forward-looking hook.)

The honest, powerful through-line: **"The hard foundation is built and real; the intelligence layer is started and has a clear path to enterprise-grade."**

---

## Recommendations

### Ideal demo flow (≈20 min)
1. **Open with the problem + scale** (2 min): the corpus numbers, the manual-pain status quo.
2. **Show the foundation** (3 min): a provider's structured profile + the verbatim clause text behind it — "we kept the actual language."
3. **Hero demo — clause standardization (22)** (5 min): pick termination or confidentiality; show standard vs outlier across the network. Let it land.
4. **Semantic search + grounded citation QA (17 → 20)** (5 min): a language question, answered with cited sources; click through to a source PDF.
5. **Amendment timeline (21)** (3 min): show a contract's full history chain.
6. **Close with the roadmap** (2 min): negotiation/renewal as the next wave; name the foundation as the moat.

### Safest demo workflow
**Clause similarity / outlier detection (22)** on a single clause family, against a curated provider. It runs on real embeddings, degrades gracefully, has no current-state or dollar claims to falsify, and is visually striking. If you demo only one thing, demo this.

### Highest-impact workflow
**Cross-portfolio clause standardization + grounded citation QA together** — "across all our contracts, here's the outlier, here's exactly what it says, and here's the source." It combines fleet-level intelligence with auditable provenance, which is the combination that reads as *enterprise*.

### Strongest enterprise narrative
*"We converted an unstructured contract archive into a governed, searchable, citable intelligence layer — the foundation a payer needs to manage its provider network at scale — and we've begun layering negotiation and renewal intelligence on top."* Emphasize foundation + provenance + roadmap.

### Best sequence of demonstrations
Scale → foundation (structure + verbatim text) → semantic search → cross-portfolio outlier analysis → grounded cited answer → amendment timeline → roadmap. (Build from "we have the data" to "we can reason across it" to "here's where it's going.") Always end a thread on a *source citation* so the last impression is trust.

### Best live questions to handle (invite these)
- *"Where did that answer come from?"* → click to the cited source PDF/page. Strongest possible moment.
- *"Is this clause standard across our network?"* → run the outlier tool live.
- *"How many contracts/providers are in here?"* → cite corpus scale (be ready to say "this phase covers the extracted set; full-corpus extraction is in progress" if pressed).
- *"Can it find language about [topic]?"* → live semantic search; works well for genuine clause topics.
- *"How is this different from ChatGPT on our PDFs?"* → the structured-model + cross-portfolio + provenance answer from §5.

### Questions to avoid (or scope honestly if asked)
- *"What's the current rate for X as of today?"* → reframe to "here's the latest extracted rate and its source"; don't claim authoritative current-state.
- *"What did amendment N change vs N-1?"* → show the timeline; don't run the rate diff live.
- *"What will renegotiating save us in dollars?"* → "that requires joining claims/utilization volume, which is on the roadmap" — honest and forward-looking.
- *"Show the 3-year rate trend."* → not available; pivot to amendment history.
- *"Does it cover every contract?"* → be honest: extracted subset now, full corpus in progress.
- *"Is the displayed 'CURRENT' guaranteed in-force?"* → "it reflects the latest extracted document; precise point-in-time state is a roadmap item."

**Golden rule:** frame every answer as *"based on the extracted contract language, with its source"* rather than *"this is the authoritative current legal state."* That single discipline keeps the demo truthful and still impressive, and converts every weakness into a credible roadmap item rather than a caught flaw.

---

*Synthesis of Phases 1–6; no new repository scan. For the underlying evidence behind each risk, see `ILLUSION_VS_REALITY_AUDIT.md` and the phase documents it references.*
