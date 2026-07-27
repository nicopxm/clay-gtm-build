# PROJECT_BRIEF.md — Clay GTM Table
**Status:** Active · **Sprint length:** 3 working days · **Owner:** Wop
**Portfolio slot:** Project 2 of 4 · **Primary track served:** GTM Engineer / RevOps

---

## 1. Business Problem (the framing that opens the write-up)
Manual account research costs SDR teams hours per day and paid databases (ZoomInfo-class) run $15k+/yr with mediocre match rates. This project demonstrates a credit-efficient enrichment system that turns a raw company list into scored, CRM-ready leads — and quantifies when to buy a tool like Clay vs. build the equivalent in n8n.

**Meta-angle (intentional):** the target list is companies currently hiring RevOps/GTM roles — the output doubles as Wop's own outbound target list.

## 2. ICP Definition (locked — do not relitigate)
- B2B SaaS, Series A or B
- 20–200 employees
- Currently has an open RevOps / GTM Engineer / Sales Ops posting (source signal: job boards)
- US-based or US-market-focused (contractor/international hiring feasible)
- Stack signals worth points: HubSpot, Clay, n8n/Zapier, Outreach/Apollo mentioned in postings

## 3. Table Architecture — FINAL (locked 2026-07-15, chat: Day 1 design)

### Column groups, in order
1. Seed data (imported): company, domain, careers_or_posting_url,
   posting_title, posting_date, stage_hint
2. Firmographic waterfall: free/cached → cheap → expensive LAST.
   Conditional: skip paid if free source filled the field.
3. ICP GATE (boolean, conjunctive AND, cost-ordered checks):
   G1. posting_title regex: RevOps|Revenue Operations|GTM Engineer|
       Sales Op(s|erations)|GTM (Strategy & )?Operations
                                                       [seed data, free]
       (amended 2026-07-15, post-seed-audit: one-time widening for the
       "GTM Strategy & Operations" title variant — observed in real
       data, in-ICP, mis-parsed by adjacency. Pattern now LOCKED.)
   G2. employee count 20–200                          [free tier only]
   G3. US-based or US-market-focused                  [seed/free only]
   G4. Series A or B                                  [cheap provider,
                                                       runs last]
   RULE: missing free data on G2/G3 = FAIL. Zero rescue lookups.
   Deliberate false-negative acceptance; report N rows failed on
   missing data as the cost of $0 wasted credits.
   Everything downstream runs ONLY if gate = pass.
4. AI column (exactly one): gtm_motion — PLG/sales-led/hybrid/unknown
   from homepage + pricing text. Strict JSON out. Does NOT feed
   scoring. Prompt saved in Project (Day 1 chat).
5. Scoring formula — 4 dimensions, weights sum to 100, deterministic
   aggregation in a formula column: icp_score = Σ(dim × weight)/100.
   Same contract as pipeline's ICP schema (docs/ICP-CONFIG.md); config
   file does not exist yet — "same scoring contract, two runtimes."

   | Dimension        | Wt | Scored how                              |
   |------------------|----|-----------------------------------------|
   | Stack match      | 35 | 25 pts per match (HubSpot, Clay,        |
   |                  |    | n8n/Zapier, Outreach/Apollo), cap 100   |
   | Hiring intensity | 25 | Tiered — see below                      |
   | Size sweet spot  | 20 | 100 if 40–120 employees; 60 if 20–39    |
   |                  |    | or 121–200                              |
   | Funding recency  | 20 | 100 if raise <12mo; 60 if 12–24mo;      |
   |                  |    | 30 if older/unknown date                |

   Hiring intensity tiers (additive, cap 100):
   - Seniority: Head/Director/VP title +50; Manager +30; other +10
   - Multiple GTM/RevOps postings open: +30
   - Posting <30 days old: +20
   Geo: NOT scored — gate criterion (no "more US than US").

6. Contact waterfall (conditional): 1 contact (Head of RevOps/Sales),
   only rows above score threshold.
7. HubSpot push: name/email/company/lead_source + icp_score (first-ever
   write to that property). Dedupe: domain exists → update, not create.
   Present mapping as current vs. planned per REALITY.md §3.

## 4. Benchmark Spec (build-vs-buy section)
- Same 20 accounts run through (a) Clay waterfall, (b) existing n8n enrichment workflow.
- Capture per system: match rate %, cost per enriched lead, elapsed time, setup effort (qualitative).
- Output: one comparison table + 3-sentence verdict on when to choose each.
- Script (if needed) runs via Claude Code against the EXISTING n8n workflow. No new workflow building.

## 5. Sprint Plan
| Day | Output | Done means |
|---|---|---|
| 1 AM | Schema + scoring designed in chat | This doc's section 3 finalized with weights |
| 1 PM | Table built in Clay, waterfall running | 100 rows imported, gate + waterfall live, screenshots taken |
| 2 AM | HubSpot push working | Leads visible in HubSpot, dedupe tested |
| 2 PM | Benchmark run | Comparison table filled with real numbers |
| 3 | Write-up + LinkedIn post | 2 pages, hiring-manager order, post drafted |

## 6. Write-up Quality Bar (hiring-manager checklist)
- [ ] Opens with the business problem, not the tool
- [ ] Waterfall ordering justified in cost terms
- [ ] At least one conditional-logic decision explained
- [ ] Scoring weights have written rationale
- [ ] Honest numbers included (match rate, cost/lead) — including what failed
- [ ] Ends in HubSpot with dedupe answered ("then what?" test)
- [ ] Build-vs-buy verdict with benchmark data
- [ ] Passes the 2-minute test: a design decision explainable without saying "Clay"

## 7. Out of Scope (PARKED — do not build)
- More than 100 rows / list expansion
- Multiple AI columns or generic "personalization" columns
- Email sequences / outreach sending (that's outbound activity, not this project)
- New n8n workflows, new infra, new tools
- Automating the list refresh (mention as "future work" in write-up, one line max)
- Perfecting provider selection beyond one reorder pass

## 8. Risks
- **Clay free-tier credit limits:** design conditionals FIRST, screenshot everything, document limits honestly if hit.
- **Clay trial expiry: 2026-07-29** — hard deadline for Day 2 + benchmark (see REALITY.md §9).
- **Scope creep:** any new idea → PARKED list in this doc, reviewed only after Day 3 ships.
- **Benchmark rabbit hole:** timebox to half a day; directionally-correct numbers beat precise ones.
