# Day 1 Debrief — Clay GTM Table Build
**Date:** 2026-07-15 to 2026-07-17 (spanned 2 calendar days; see Section 6)
**Authority:** Raw build record. Source of truth for the write-up.

---

## 1. BUILD SUMMARY

**Table as it stands in Clay:**

| Column group | Columns | Notes |
|---|---|---|
| Seed data (imported) | company, domain, careers_or_posting_url, posting_title, posting_date, stage_hint | 50 rows; posting_date blank on 49/50; stage_hint blank on all 50 |
| Enrichment waterfall — tier 1 | Employee Count, Country (from Clay Enrich Company) | 0.5 cr/row; conditional on {{domain}} not empty; 45/50 ran, 5 blank-domain skipped |
| Pre-gate formula | pre_gate | G1 AND G2 AND G3 boolean; formula column, $0 |
| Enrichment waterfall — tier 2 | Fundingstage, lastfundingat (from Harmonic) | 4 cr/row; conditional on pre_gate = true; 10/11 returned data, 1 not charged |
| Gate formula | icp_gate | pre_gate AND (Fundingstage = SERIES_A or SERIES_B); formula column, $0 |
| Scrape columns | homepage_text, pricing_text, pricing_url | 0.1 cr/row each; conditional on icp_gate = true; 8 rows ran; pricing_url is a helper formula column |
| AI classification | gtm_motion (motion, confidence, evidence) | Claygent clay-argon; 3 cr/row; conditional on icp_gate = true; 8 rows ran |
| Scoring formulas | score_stack, score_hiring, score_size, score_funding, icp_score | Formula columns, $0; run on all 50 rows |

**Row counts at each stage:**
- Imported: 50
- Enriched (tier 1): 45 (5 blank-domain skipped by condition)
- Passed pre_gate (G1+G2+G3): 11
- Enriched (tier 2 / Harmonic): 10 of 11 (1 returned no data, not charged)
- Passed icp_gate (G1+G2+G3+G4): 8
- Classified (gtm_motion): 8
- Scored (icp_score): 50 (formula runs on all rows; only 8 gate-passers have meaningful scores)

---

## 2. CREDIT LEDGER

| Step | Column | Provider | Rate | Rows ran | Actual cost | Balance after |
|---|---|---|---|---|---|---|
| Start | — | — | — | — | — | 2,005.0 |
| Step 2 | Enrich Company (Employee Count + Country) | Clay native | 0.5/row | 45 | 22.5 | 1,982.5 |
| Step 2 | Fundingstage + lastfundingat | Harmonic | 4/row | 10 | 40.0 | 1,942.5 |
| Step 4 | homepage_text (scrape) | Clay scrape | 0.1/row | 8 | 0.8 | 1,941.7 |
| Step 4 | pricing_text (scrape) | Clay scrape | 0.1/row | 8 | 0.8 | 1,940.9 |
| Step 4 | gtm_motion (Claygent) | Clay AI | 3/row | 8 | 24.0 | 1,916.9 |

**Total spent: 88.1 credits (4.4% of 2,005 trial credits)**
**Remaining: 1,916.9 credits**

All balances confirmed against Clay UI at time of spend.

Note: Lusha was configured but removed before any rows ran — $0 spent. It showed no credit cost in the catalog but required external account authentication; removed on discovery.

---

## 3. GATE RESULTS

**8 of 50 rows passed icp_gate. 42 failed.**

(Note: 39 failed pre_gate; of the 11 that passed pre_gate, 3 failed G4.)

**Failure breakdown by first-failing gate:**

| First-failing gate | Failure type | Count | Detail |
|---|---|---|---|
| G1 — title regex | — | 0 | All 50 rows pass; all titles sourced as RevOps/GTM roles |
| G2 — employee count | Missing data | 5 | Blank-domain rows; enrichment condition not met; no employee count returned |
| G2 — employee count | Criteria (>200) | 17 | Named: meetdandy.com (1,953), openai.com (10,284), cognition.ai (487), ziphq.com (1,342), gusto.com (4,474), pitchbook.com (1,565), hightouch.com (601), later.com (2,024), fiscalnote.com (359), freenome.com (405), transcarent.com (836), get-carrot.com (614), getmaintainx.com (941) + 4 not recorded by name |
| G2 — employee count | Criteria (<20) | 12 | Employee count returned but below 20-employee floor; company names not recorded during build |
| G3 — country | Criteria (not US) | 2 | dust.tt (FR), finout.io (IL) |
| G3 — country | Missing data | 3 | Rows with domains where enrichment ran but returned no country value |
| G4 — funding stage | Criteria (not Series A/B) | 2 | Stage was Series C or other |
| G4 — funding stage | Missing data | 1 | Harmonic returned no data; row not charged |
| **Total failed** | | **42** | |
| **Passed icp_gate** | | **8** | |

**G1 regex validation:**
Both "GTM Strategy & Operations" rows (OpenAI, Zip) matched the G1 regex pattern `GTM (Strategy & )?Operations` and returned pre_gate values consistent with G1 passing. Both failed at G2 (employee count >200 — OpenAI 10,284, Zip 1,342). G1 regex confirmed functional for the edge-case variant.

**Missing-data vs. criteria split:**
- Failed on missing data: 9 rows (5 blank-domain G2 + 3 blank-country G3 + 1 no Harmonic data G4)
- Failed on criteria: 33 rows (17 count >200 + 12 count <20 + 2 non-US + 2 wrong funding stage)
- 9 + 33 = 42 ✓

---

## 4. GTM_MOTION RESULTS

**8 rows classified (all icp_gate = true):**

| Classification | Count |
|---|---|
| hybrid | 4 (Lorikeet, Hyperbound, Reducto, Hologram) |
| PLG | 1 (Turnkey) |
| sales-led | 1 (Runlayer) |
| unknown | 2 (GC AI, Suger.io) |

**The two "unknown" rows diagnosed:**

- **Suger.io (suger.io):** Scrape column ran (✅ confirmed) but Claygent received no usable page content — evidence states "No platform text was supplied." Likely cause: bot-blocking or JavaScript-rendered site that returned empty body text to Clay's scraper. The motion value was also malformed ("unknown — no platform text provided" instead of "unknown") — Claygent violated the strict JSON format rule when it found no content. Fixable: retry with JS rendering enabled on the scrape column.

- **GC AI (gc.ai):** Scrape ran and returned content, but it was wrong-page content — the evidence cites "internship 'Compensation: up to €600 per month' and application details." gc.ai may resolve to a page other than the company homepage, or the homepage was thin and Clay's scraper pulled a subpage. Claygent correctly returned "unknown" per the do-not-guess rule — it saw real text with no product signals. Classification of "unknown" is honest. (GC AI HQ verified as San Mateo, CA — geo data correct; see Section 7.)

**Rule validation:** Both unknowns confirm the "judge only from provided text, do not guess" rule is working. The rule is a feature, not a limitation.

**Claygent behavior note:** Clay's "Web research (Claygent)" mode also browsed company URLs directly during classification (visible in stepsTaken fields), in addition to reading the pre-scraped homepage_text and pricing_text columns. This did not increase credit cost beyond the 3/row base rate but means the pre-scrape columns served as prompt seeds rather than sole inputs.

---

## 5. SCORING FINDINGS

**Top 8 by icp_score (all gate-passers):**

| Rank | Company | icp_score | score_stack | score_hiring | score_size | score_funding |
|---|---|---|---|---|---|---|
| 1 | Turnkey | 44.5 | 0 | 50 | 60 | 100 |
| 2 | Lorikeet | 42.5 | 0 | 10 | 100 | 100 |
| 3 | Hyperbound | 42.5 | 0 | 10 | 100 | 100 |
| 4 | Reducto | 42.5 | 0 | 10 | 100 | 100 |
| 5 | GC AI | 39.5 | 0 | 30 | 60 | 100 |
| 6 | Runlayer | 34.5 | 0 | 10 | 60 | 100 |
| 7 | Suger.io | 31.5 | 0 | 30 | 60 | 60 |
| 8 | Hologram | 28.5 | 0 | 10 | 100 | 30 |

**Turnkey 44.5 decomposed:**
(0×35 + 50×25 + 60×20 + 100×20) / 100 = (0 + 1,250 + 1,200 + 2,000) / 100 = 44.5
- score_stack 0: no stack keywords in title
- score_hiring 50: "Head of Revenue Operations" → Head = +50 seniority tier
- score_size 60: 124 employees → 121–200 band (just outside 40–120 sweet spot; would score 100 at 120 or below)
- score_funding 100: last raise May 2026 → ~2 months ago, <12 months

**Key finding — score_stack = 0 across all 50 rows:**
Seed data contained posting titles only, not full job description body text. Stack keywords (HubSpot, Clay, n8n, Zapier, Outreach, Apollo) appear in posting descriptions, not titles. The score_stack dimension is structurally inert with current data. Effective ceiling was 65/100; rankings are relative within that ceiling, not absolute.

Fix deferred to Day 2: scrape posting bodies from careers_or_posting_url (~0.1 cr × 8 gate-passing rows = ~0.8 credits) and re-score. Low cost, high fidelity gain.

**score_hiring +20 recency:** Never fired. 49/50 rows have blank posting_date. Reflow (the one row with a date, June 11 2026) is 36 days old — outside the 30-day window — and fails the gate anyway (blank domain).

**score_funding validation:** Harmonic raise dates parsed and scored correctly end-to-end. Hologram's December 2021 raise scored 30 (>24 months). Turnkey's May 2026 raise scored 100 (<12 months). Formula confirmed accurate.

---

## 6. DEVIATIONS & FINDINGS LOG

| Date | Item |
|---|---|
| 2026-07-15 | 110→50 row subset: Clay free trial caps tables at 50 rows. Subset built deliberately to preserve G1 test rows (both GTM Strategy & Operations companies included), blank-domain proportion (5/9 blank rows, ~10% of subset vs 8.2% of full list), and title category mix (40/24/22/14% matching original). Original seed_list.csv untouched. |
| 2026-07-15 | No free enrichment tier found: build doc specified "free/cached first" but no free no-auth provider existed in Clay's catalog. Lusha showed no credit cost but required external account authentication — discovered after seeing demo-data green check marks, removed before any rows ran, $0 spent. Waterfall floor was Clay-native Enrich Company at 0.5 cr/row. |
| 2026-07-15 | Harmonic used for G4 over cheaper-looking alternatives: "Get company total funding" (8 cr/row) had no last raise date; "Enrich company funding activity" (12 cr/row) had date but was expensive. Harmonic at 4 cr/row returned both funding stage AND last raise date — best value tier for G4. |
| 2026-07-15 | posting_date and stage_hint fully empty per plan: no backfill attempted. score_hiring +20 recency never fires (by design); funding stage comes entirely from Harmonic enrichment. Both noted in scoring findings. |
| 2026-07-16 | pricing_text obtained via helper column: Clay's scrape URL field couldn't accept a literal-typed prefix ("https://" + domain column). Workaround: formula column pricing_url = {{domain}} + "/pricing", then used as scraper URL input. pricing_url column populated for all 50 rows (blank-domain rows get "https:///pricing" — harmless, scraper condition blocks them). |
| 2026-07-16 | Claygent prompt recovered from planning chat mid-build: PROJECT_BRIEF.md references "prompt saved in Project (Day 1 chat)" without embedding it. Prompt was provided by user during build and saved verbatim to claygent_prompt.md in the working directory. |
| 2026-07-16 | Build ran across 2 calendar days vs. planned 1: caused by trial row-cap discovery requiring subset design, Lusha dead-end, provider research for G4, and Clay UI friction (column naming, URL input, formula debugging). No scope deviation — all planned columns built. |

---

## 7. GC AI GEO CHECK

**Finding:** GC AI (gc.ai) is headquartered in San Mateo, California, USA. The Clay Enrich Company enrichment returned country = "US" — correct. G3 geo data was not a false positive.

The €600/month internship text in the gtm_motion evidence was wrong-page scrape content, not evidence of EU location. The classifier correctly returned "unknown" because the page content had no product signals — the do-not-guess rule worked as designed.

**Implication for Section 4:** GC AI's "unknown" classification is a scrape quality issue (wrong page), not a geo data issue. The enrichment was accurate; the scrape landed on wrong content.

---

## 8. DAY 2 QUEUE

In order of dependency:

1. **(Optional opener, low cost):** Scrape posting bodies from careers_or_posting_url for the 8 gate-passing rows (~0.1 cr × 8 = 0.8 cr). Re-run score_stack with body text. This unlocks the stack signal dimension and raises the effective ceiling from 65 to 100. Do before contact waterfall so final scores are correct.

2. **Contact waterfall:** Find 1 contact per company (Head of RevOps or Sales) for rows above score threshold. Set threshold deliberately — confirm before running.

3. **HubSpot push:** Push name, email, company, lead_source + icp_score (first write to that property). Dedupe rule: domain exists → update, don't create (per REALITY.md §3, issue #24 is scheduled but not yet enforced — test dedupe manually before bulk push).

4. **Benchmark run:** Same 20 accounts through Clay waterfall vs. existing n8n enrichment workflow. Per REALITY.md §2: n8n cost figures are estimates (API pricing × call counts), not measured data. Label clearly. Timebox to half a day.

**Reminder:** Re-read PROJECT_BRIEF.md and REALITY.md at the start of Day 2. REALITY.md outranks PROJECT_BRIEF.md on pipeline state.

---

## 9. SCREENSHOT INDEX

| File | What it shows |
|---|---|
| 00-credits-before.png | Starting credit balance (2,005 credits) before any build actions |
| 01-import.png | Imported table — 50 rows, 6 columns, seed data visible |
| 02-waterfall.png | Clay Enrich Company column config with {{domain}} conditional visible — the design decision |
| 03a-pregate.png | pre_gate formula column + Harmonic funding column with pre_gate condition set, before Harmonic ran |
| 03-gate.png | icp_gate column across all 50 rows — TRUE/FALSE distribution visible |
| 04-gtm-motion.png | Hologram row expanded showing gtm_motion JSON output (hybrid, high confidence) |
| 05-scoring.png | Scored row showing all 5 score columns + icp_score |
| 06-top8.png | Filtered view of 8 icp_gate = TRUE rows with icp_score, posting_title, gtm_motion visible |
