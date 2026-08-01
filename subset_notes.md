# Subset Notes — seed_list_50.csv

**Why this file exists:** Clay free trial caps tables at 50 rows. Rather than importing the first 50 rows, a deliberate subset was built to preserve the statistical properties of the full 110-row list.

## Selection logic

**Total selected:** 50 of 110 rows. Original `seed_list.csv` is untouched.

### Mandatory rows (2)
Both "GTM Strategy & Operations" postings are included — they are the G1 regex test case (the `GTM (Strategy & )?Operations` branch of the pattern). Without them, the gate test for that title variant has no coverage.
- Row 63: OpenAI — "GTM Strategy & Operations Lead - BDR"
- Row 65: Zip — "GTM Strategy & Operations Senior Manager (multiple postings)"

### Blank-domain rows (5 of 9)
5 of the 9 blank-domain rows are included (≈10% of subset vs 8.2% of full list — proportional).
These rows fail enrichment at the waterfall step and die at G3 (no geo signal) or G2 (no employee count). They represent the honest cost of the $0-wasted-credits rule: missing free data = FAIL, no rescue lookups.

| Company | Title | Note |
|---------|-------|------|
| Palette Media | Revenue Operations Manager | blank domain |
| The Global Talent Co. | GTM Engineer / AI Engineer | blank domain |
| Iru | Manager, GTM Operations | blank domain |
| Cassi Home | Revenue Operations Lead | blank domain |
| Reflow | GTM / Revenue Operations (contract) | blank domain; **only row with a posting_date** (June 11 2026) |

Blank-domain rows excluded from subset: telli, Prox, Propel, Trase Systems.

### Remaining 43 rows
Sampled proportionally by title category using evenly-spaced draws from each pool. No cherry-picking for gate passage.

| Category | Full list | Subset |
|----------|-----------|--------|
| Revenue Operations | 44 (40%) | 20 (40%) |
| GTM Engineer | 26 (24%) | 12 (24%) |
| Sales Operations | 24 (22%) | 11 (22%) |
| GTM Operations | 16 (15%) | 7 (14%) |

## posting_date and stage_hint

**Both columns are blank on 49 of 50 rows.** Reflow (row 110) is the single exception with `posting_date = "June 11 2026"` — preserved because it is real seed data, not backfilled.

Write-up implications:
- `score_hiring` +20 recency bonus fires only for Reflow (if it passes the gate). On all other rows it does not fire — **by design, not a data gap**. Do not backfill or scrape dates.
- `score_funding` relies on the funding enrichment waterfall (Step 2), not seed data. `stage_hint` being blank on all rows is expected; funding stage comes from the paid provider in the waterfall.
- "109 rows with no posting_date" from the build doc becomes "49 of 50 rows" in this subset.

## Waterfall tier reality (adapted from build doc)

**Build doc says:** free/cached tier first, then cheap paid fallback.
**What the trial actually offers:** Lusha (no credit shown) requires external account auth — unusable without credentials. The Clay-native "Enrich Company" at 0.5 cr/row is the cheapest available no-auth option.

**Actual waterfall structure built:**
- Tier 1 (base): Clay-native Enrich Company — 0.5 cr/row, returns employee count + country in one call
- Tier 2 (fallback): not needed — 100% fill rate on all 45 rows with domains
- G4 funding data: separate enrichment, runs last (conditional on pre-gate passing)

**Credit spend — enrichment waterfall:**
- Starting balance: 2,005 credits
- After Enrich Company run: 1,982.5 credits (22.5 spent)
- 45 rows × 0.5 cr = 22.5 cr; 5 blank-domain rows skipped by `{{domain}}` condition = $0

**Fill rates:**
- Employee count: 45/45 (100% on rows with domains)
- Country: 45/45 (100% on rows with domains)
- Blank-domain rows: 5 skipped by condition, 0 credits charged

Write-up talking point: "The conditional run saved 5 × 0.5 = 2.5 credits — small here, meaningful at scale. The design decision is the pattern, not the dollar amount."

## Gate results (Step 3)

**icp_gate: 8 of 50 pass (16%)**

| Gate | Condition | Notes |
|------|-----------|-------|
| G1 | posting_title regex | All 50 rows pass — all titles sourced as RevOps/GTM roles |
| G2 | Employee count 20–200 | Main killer: 17 rows >200 (OpenAI 10k, Gusto 4.4k, Dandy 1.9k, etc.) |
| G3 | US-based | 2 non-US (Dust=FR, Finout=IL); 3 rows enriched but returned no country |
| G4 | Series A or B | 2 rows Series C or other; 1 row no Harmonic data returned |

**Failure breakdown (corrected 2026-08-01 — see note below):**
- Failed on missing data: **9 rows** (5 blank-domain at G2; 3 no country returned at G3; 1 no Harmonic data at G4)
- Failed on criteria: **33 rows** (17 headcount >200; 12 headcount <20; 2 non-US; 2 wrong funding stage)
- 9 + 33 = 42 ✓

> **Correction note (2026-08-01):** this section originally read "~8 missing data / ~34 criteria," with blank-country attributed as 2 rows rather than 3. Those were in-build estimates (marked with tildes) written before the per-row failure attribution was completed. `day1_debrief.md` §3 is the authoritative count — it attributes every one of the 42 failures to a first-failing gate and reconciles to 42. Corrected here so the two files agree; **day1_debrief.md outranks this file on gate results.**

**Credit spend to reach gate:**
- Clay Enrich Company (G2+G3 data): 22.5 cr
- Harmonic (G4 data, conditional on pre_gate): 40.0 cr
- Total enrichment spend: 62.5 cr
- Balance after Step 3: 1,942.5 cr

Write-up line: *"8 of 50 rows survived all four gates. 9 failed on missing data at $0 in rescue lookups — the cost of the deliberate false-negative policy."*

## Credit ledger (reconciled against Clay UI)

| Step | Column | Provider | Rate | Rows ran | Actual cost | Balance after |
|------|--------|----------|------|----------|-------------|---------------|
| Start | — | — | — | — | — | 2,005.0 |
| Step 2 | Enrich Company (employee count + country) | Clay native | 0.5/row | 45 (5 blank-domain skipped) | **22.5** | 1,982.5 |
| Step 2 | Funding stage + last raise date | Harmonic | 4/row | 10 (1 of 11 pre_gate rows returned no data, not charged) | **40.0** | 1,942.5 |
| Step 4 | Homepage scrape | Clay scrape | 0.1/row | 8 (icp_gate=true only) | **0.8** | 1,941.7 |
| Step 4 | Pricing page scrape | Clay scrape | 0.1/row | 8 (icp_gate=true only) | **0.8** | 1,940.9 |
| Step 4 | gtm_motion (Claygent) | Clay AI | 3/row | 8 (icp_gate=true only) | **24.0** | 1,916.9 |

**Confirmed actual balances (all UI-verified):** 2,005.0 → 1,982.5 → 1,942.5 → 1,941.7 → 1,940.9 → 1,916.9
**Total spent: 88.1 credits. Remaining: 1,916.9 credits.**

### Step 5–6 notes (zero additional credits)
- score_stack = 0 for all rows: seed data contains posting titles only, not full job descriptions. Stack keyword detection (HubSpot, Clay, n8n, Zapier, Outreach, Apollo) requires full posting body text.
- score_hiring +20 recency never fires: 49 of 50 rows have blank posting_date; Reflow has June 11 2026 (36 days old, outside 30-day window) and fails the gate anyway.
- Top 8 gate-passers: Turnkey 44.5, Lorikeet 42.5, Hyperbound 42.5, Reducto 42.5, GC AI 39.5, Runlayer 34.5, Suger.io 31.5, Hologram 28.5
- Screenshot named 06-top8.png (not top10) — 50-row subset produced 8 gate passers, not 10.
- gtm_motion quality flags: Reducto evidence cites external research articles (Claygent web-research mode); Suger.io returned malformed motion value "unknown — no platform text provided" (scrape failed).

Note: Lusha was attempted as a free base tier but requires external account auth — removed before any rows ran, $0 spent.

## What this means for write-up stats
All pass/fail rates, match rates, and score distributions in the write-up are based on this 50-row subset. State explicitly: *"Built on a 50-row deliberate subset (Clay free-tier cap); full-list proportions preserved."*
