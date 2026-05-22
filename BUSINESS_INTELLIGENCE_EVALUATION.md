# Business & User-Facing Intelligence Layer — Evaluation (Phase 5)

*Code-grounded review of the business-intelligence apps (18 contract comparison, 21 amendment impact, 22 clause similarity, 24 renewal intelligence, 25 negotiation intelligence) and the query workflows (19/20/23), judged from the perspective of the people who would actually use them. Citations are `notebook:line`.*

---

## Thesis

The intelligence layer **asks exactly the right questions** — peer benchmarking, "what should we propose," "what changed between amendments," "is this clause standard," "what's expiring." The capability map matches how provider-contract teams actually think. But every app shares the same three limits that keep it at **analyst-toy** rather than **decision-tool** grade:

1. **No money.** There is no claims, utilization, or membership volume anywhere in the platform. Negotiation "savings," rate "impact," and renewal "risk" are all expressed per-rate-line or as percentile positions — never as annualized dollars. A negotiator cannot see "this proposal saves $2.3M"; only "this category is at the 78th percentile." That is the single biggest gap between these tools and what the business needs.
2. **Built on the unreliable current-state model.** Everything sits on `tbl_genie_rates_current` / `tbl_genie_provider_profile`, which carry the fabricated program/network, document-level supersession, and coarse `rate_category` problems from Phases 2–4. The analytics are only as trustworthy as that substrate, and it is not trustworthy enough for rate decisions.
3. **Not products — notebooks.** Every output is a `print()` to a console in an interactive notebook you edit and run by hand. There are no persisted tables, dashboards, alerts, scheduled jobs, exports, or APIs. "Proactive renewal monitoring" (24) is a notebook someone must remember to run; "alerts" are printed lines.

---

## Per-capability evaluation

### Negotiation intelligence (25)
The most developed app: peer cohort → rate benchmark → rate proposal → LLM brief, with modes `peers|benchmark|proposal|brief`. The framing is excellent and the LLM brief is a genuinely useful artifact. But:
- **Benchmarking is at `rate_category` granularity** (`25:225-240`), which lumps Med/Surg, ICU/CCU, NICU, and Rehab into one "inpatient_per_diem" distribution. Those services differ 3–5×, so a provider's "average per-diem percentile" is an apples-to-oranges number. Real benchmarking must be service-line specific (ICU vs ICU).
- **Proposals carry no volume** (`25:358-368`): "savings_per_unit = current − target" is per rate line, not annualized spend. The advertised "savings projection" cannot be a real dollar figure.
- **Peer cohorts are structural, not market-based** (`25:154-179`): payment type + category overlap + portfolio size + per-diem proximity. A rural 50-bed and an academic medical center can be "peers" if their rate counts match. Geography, acuity, system affiliation, and market power are absent.
- **"Leverage points" aren't from contract terms.** The brief prompt receives profile + benchmark + proposal + peers only (`25:453-475`); it never sees termination/exclusivity/competitive language, so leverage is LLM-inferred from rate position, not contractual reality.
- The `rate_numeric > 0 AND < 100000` filter (`25:217`) silently drops formula/percentage rates, capitation, and the most expensive transplant case rates from the distribution.

### Reimbursement comparison (18 + 25 benchmark)
18 is a static two-provider side-by-side across profile/rates/financial/clauses/amendments/programs — set `PROVIDER_A`/`PROVIDER_B` and run all cells (`18:30-34`). The dimensional coverage is good, but it's hardcoded, two-provider-only, and descriptive. The deeper problem for reimbursement analysts: **rates are shown stripped of the operational conditions that determine actual payment** — stop-loss attachment, outlier caps, carve-outs, and lesser-of/chargemaster rules live in `tbl_contract_financial_protections` and `documents_master` and are not joined into the rate view. A per-diem without its stop-loss and carve-out logic is not a reimbursement model; it's a sticker price.

### Renewal intelligence (24)
Expiration tracker + rate-trend + amendment velocity + composite risk score + alerts. The portfolio risk-ranking idea is sound, but:
- **Expiration is estimated, not read** (`24:128`): `estimated_expiration = latest_amendment_date + contract_term_years`, defaulting to "+1 year" when term is unknown (`24:132`). Every "days remaining" and urgency flag is a guess — and many BSC agreements are evergreen/auto-renewing, where term-expiration isn't even the relevant event (the rate-renegotiation date is). The tool mismodels how evergreen provider contracts actually renew.
- **"Rate escalation" is not escalation.** `get_rate_trends` computes how far a provider's average sits *above the network* (`24:232-233`) — a cross-sectional level, relabeled as escalation. The advertised "annualized % change over time" is not implemented. So the 20% "escalation" weight in the risk score is a level metric in disguise.
- **Risk weights are arbitrary and unvalidated** (`24:403-409`), and proximity (40%) dominates while being a guess. Directionally interesting, quantitatively unreliable.

### Provider benchmarking (25 benchmark, 24 rate-trends)
Same engine, same limits: category-level aggregation, no spend weighting, no service-line normalization, no risk-adjustment for case mix or acuity, and a distribution polluted by fabricated program/network and dedup residue.

### Semantic clause comparison (22)
Conceptually the strongest app: find-similar / find-outliers / NxN matrix over the structured VS index, answering "is our termination clause standard or unusual?" (`22:5-21`). This is exactly what legal teams want. Caveats: it inherits the Phase-4 flattening and document-level currency flag; and **embedding cosine similarity is a proxy for textual resemblance, not legal equivalence** — "30 days notice" and "90 days notice" read as highly similar but are materially different, so the outlier ranking can call a substantively different clause "standard." Useful as a triage lens, not a compliance verdict.

### Operational intelligence
**Essentially absent.** Timely-filing windows, clean-claim periods, DOFR, delegated activities, credentialing, chargemaster governance, and notice periods exist only as flat text fields/clauses (Phase 3). No app surfaces them as tracked, dated, actionable operational intelligence. Operational teams are effectively unserved.

### Amendment impact (21) — right idea, wrong data source
Side-by-side amendment diff (rates added/removed/modified, clause topic changes, exhibits replaced) is precisely what contract managers want. But `diff_rates` reads `tbl_genie_rates_current` (`21:110-122`) — the **current-only, deduped** table. Comparing two *historical* amendments can't work, because superseded rates aren't in the current snapshot under their original `source_filename`; an older document returns almost nothing, so the diff reports most rates as "added in B." It should read `tbl_contract_rates_all` (which preserves superseded rows). And clause diffs are **topic-presence only** (`21:211-216`) — they detect that a topic appeared/disappeared, not that termination *language changed* (same topic, different text counts as "unchanged"). The most important amendment signal — modified clause wording — is invisible.

### User query workflows (19/20/23)
Covered in the Phase-4 audit: intent routing (19), citation RAG (20), conversational QA (23). For business users specifically, the issues are that each is a separate function with its own provider-name `ILIKE`/substring matching and no shared entity resolution or session, so a user's "Garfield" may resolve differently across tools, and answers are unverified-grounding text (Phase 4).

---

## Per-persona verdict

**Provider negotiators** — *Partially served, not decision-ready.* They get peer benchmarks, percentile positioning, rate proposals, and an LLM brief (25), plus side-by-side comparison (18) and clause-outlier checks (22). What's missing is what they'd actually walk into a negotiation with: **dollar impact** (spend × rate), **service-line-level benchmarks**, **concession/scenario modeling** ("if we give 3% on ICU, what's the net?"), **leverage from contract terms** (exclusivity, termination, competitor positioning), and **history** ("what did we agree last cycle and why"). The tool tells them where a provider sits; it can't tell them what a move is worth.

**Contract managers** — *Underserved.* Renewal monitoring (24) and amendment impact (21) target their core jobs, but expiration is guessed, the amendment diff is built on the wrong table, and nothing is persisted or scheduled. They have no obligation/deadline calendar, no contract-of-record with reliable lifecycle dates, and no portfolio worklist with ownership. The pieces of a contract-management cockpit exist as disconnected notebook prints.

**Reimbursement analysts** — *Poorly served.* Rates are decoupled from the stop-loss/carve-out/lesser-of logic that determines payment, aggregated at the wrong granularity, polluted by fabricated program/network, and never joined to claims/utilization. They cannot model actual reimbursement, compare effective vs contracted payment, or quantify the spend behind a rate.

**Legal analysts** — *Best served, with caveats.* Clause similarity/outlier detection (22) and grounded QA (23) genuinely help triage "is this language standard / what does the contract say." But clause currency is a document-level heuristic, there's no clause-level amendment lineage, similarity ≠ legal equivalence, and there's no obligation/risk extraction. Good for exploration, not for authoritative legal-state answers.

**Operational teams** — *Effectively unserved.* No operational-obligation tracking, no DOFR/delegation views, no compliance-deadline workflow.

---

## What important workflows are missing

- **Dollar-impact everywhere** — integrate claims/utilization/membership so every benchmark, proposal, and risk score is spend-weighted and expressed in annualized dollars.
- **Provider 360 / relationship-of-record** — one unified view per facility (current rates with conditions, lifecycle dates, obligations, amendment history, negotiation history, benchmarks) instead of five disconnected notebooks.
- **Obligation & deadline calendar** — timely-filing, notice periods, renewal/renegotiation triggers, delegated-activity oversight, surfaced as tracked, owned, alertable items.
- **Negotiation scenario workspace** — model concessions, see net spend impact, generate proposal + redline + approval trail, capture outcomes for next cycle.
- **Audit / point-in-time** — "what reimbursement and terms governed facility F on date D" (depends on the Phase-3 state layer).
- **Coverage/confidence surfacing** — only 34% of the corpus is extracted (Phase 3); no app tells a user when a provider's data is incomplete, so absence reads as fact.
- **Persistence, scheduling, alerting, export** — turn run-on-demand prints into monitored, deliverable products.

---

## Proposals

**Better business abstractions — organize around entities teams think in, not notebooks.** Five durable objects: **Provider Relationship** (facility 360), **Contract** (lifecycle + dates + obligations), **Fee Schedule** (spend-weighted, service-line granular, with conditions), **Obligation** (dated, owned, alertable), **Negotiation Cycle** (scenarios, proposals, outcomes). Every app becomes a view onto these, not a standalone script.

**Better intelligence views.** (1) **Provider 360** — current rates *with* stop-loss/carve-outs, lifecycle dates, open obligations, amendment history, peer position, negotiation history, and a coverage/confidence badge. (2) **Spend-weighted benchmark** — service-line-level percentiles, risk/acuity-adjusted peers, expressed as $ impact. (3) **Renewal & obligation calendar** — real dates, ownership, persisted alerts. (4) **Clause standardization library** — standard-vs-variant by clause family, with materiality (term values), not just cosine. (5) **Amendment ledger** — correct historical diffs (from `tbl_contract_rates_all`) including language-level changes.

**Better semantic serving models.** Put a **governed metrics/semantic layer** over the Phase-3 point-in-time state API so "current rate," "peer cohort," "annualized spend," "expiration," and "in force" have one definition shared by every tool and dashboard. Add a **service-line taxonomy** (controlled vocabulary mapping raw service categories to comparable lines) and a **spend join** to claims. This is what makes the numbers trustworthy and consistent across surfaces.

**Better workflows.** End-to-end **negotiation cycle**: benchmark → scenario model (spend impact) → brief → proposed redline → approval → executed amendment → monitor. **Renewal pipeline**: detect → prioritize (real dates + spend) → assign owner → alert → track to closure. **Audit lookup**: point-in-time state with provenance. Each is a stateful, multi-step workflow, not a single print.

**Better enterprise UX abstractions.** One entry point (search a provider/contract → land on the 360), **role-based views** (negotiator / manager / reimbursement / legal / ops see different default lenses on the same truth), **narrative + drill-down + provenance** (every number traces to source and as-of date), **coverage/confidence always visible**, **human-in-the-loop** for anything that proposes a number or a redline, and **deployed, scheduled, exportable** surfaces (Databricks Apps / dashboards / Genie spaces) instead of notebooks.

---

## What it gets right

The **question framing is genuinely strong** — these are the real questions provider-contract teams ask, and having peer benchmarking, amendment diffing, clause-standardization, renewal risk, and a negotiation brief all attempted in one platform is the correct ambition. The **analytical scaffolding is reasonable** (percentiles, Jaccard peer similarity, amendment velocity, composite scoring, cosine clause similarity), and the **LLM briefs/summaries add real narrative value**. As a demonstration of "what contract intelligence could look like," it is persuasive.

## What it fundamentally misses

**Decision-grade outputs.** Without spend/volume there are no dollars; without service-line granularity benchmarks are apples-to-oranges; without the operational conditions joined in, rates aren't reimbursement; without persistence/scheduling these aren't products; and because they sit on the unreliable current-state model, the numbers can't yet be trusted for real negotiations or renewals. The layer demonstrates the right questions but cannot yet deliver answers a negotiator, manager, or analyst would stake a decision on.

## What should evolve next

1. **Integrate claims/utilization/membership** and make every metric spend-weighted in dollars — the highest-value single change.
2. **Build the service-line taxonomy** and re-benchmark at service granularity.
3. **Join operational conditions** (stop-loss, carve-outs, lesser-of) into the rate/reimbursement views.
4. **Fix amendment impact** to read `tbl_contract_rates_all` and add language-level clause diffs.
5. **Consolidate into a Provider 360** over a governed semantic layer; add coverage/confidence.
6. **Deploy** as scheduled, persisted, role-based apps with real alerts (depends on, and motivates, the Phase-3 state layer).

## What enterprise legal-AI / payer-contracting systems would do differently

Mature provider-contracting platforms anchor everything to **spend**: every clause, rate, and renewal is weighted by the dollars flowing through it, so prioritization is automatic and proposals are quantified. They maintain a **contract-of-record with reliable lifecycle dates and an obligation calendar**, not estimated expirations. They benchmark at **service-line / code granularity with acuity and market adjustment**, often against external market data, not just internal averages. They treat **clause analysis as materiality-aware** (extracting the actual term values — notice days, caps, percentages — and comparing those, not just embedding text). They run **negotiation as a workflow** with scenario modeling, approval, redline generation, and outcome capture feeding the next cycle. And they ship **governed, role-based, deployed applications** with audit trails and confidence/coverage surfaced — never analyst notebooks as the delivery vehicle. The current platform has the analytical ideas; enterprises operationalize them on a trustworthy, spend-aware, deployed foundation.

---

*Method note: grounded in notebooks 18, 21, 22, 24, 25 (and 19/20/23 from Phase 4). Specific defects — category-level benchmarking, missing volume, estimated expiration, mislabeled escalation, and the amendment-diff reading the current-only table — were verified directly in the source.*
