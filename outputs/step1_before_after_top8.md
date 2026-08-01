# Step 1 — Before/After: title-only vs. posting-body scoring

**Provenance:** reconstructed 2026-08-01 from the scores recorded in `day1_debrief.md` §5 (v1 dimension scores) and `day2_debrief.md` Step 1 (v2 stack scores and threshold decision). `day2_debrief.md` referenced this file at Step 1 close but it was never written to disk; rebuilt here so the reference resolves. Every v2 total below is recomputed from its dimension scores rather than copied — the arithmetic is shown so it can be checked against either debrief.

Formula, unchanged between v1 and v2 — only the stack input changed:

```
icp_score = (score_stack × 35 + score_hiring × 25 + score_size × 20 + score_funding × 20) / 100
```

---

## The change

Day 1 scored `score_stack` against posting **titles**, because that's all the seed data contained. Stack keywords (HubSpot, Clay, n8n, Zapier, Outreach, Apollo) live in posting **bodies**. Result: `score_stack = 0` on all 50 rows, capping the effective ceiling at 65 of 100.

Step 1 scraped the posting bodies for the 8 gate-passers (1.5 cr total) and re-scored. Three of the eight postings were still live; five had closed in the ~7 days since seed collection and fell back to title-only scoring.

---

## Before / after

| Company | Posting | v1 stack | v2 stack | v1 score | v2 score | Δ | v1 rank | v2 rank |
|---|---|---|---|---|---|---|---|---|
| Hyperbound | live | 0 | 75 | 42.5 | **68.75** | +26.25 | 3 | **1** |
| Hologram | live | 0 | 75 \* | 28.5 | **54.75** | +26.25 | 8 | **2** |
| Turnkey | closed | 0 | 0 | 44.5 | 44.5 | — | 1 | 3 |
| Lorikeet | closed | 0 | 0 | 42.5 | 42.5 | — | 2 | 4 |
| Reducto | closed | 0 | 0 | 42.5 | 42.5 | — | 4 | 5 |
| Suger.io | live | 0 | 25 | 31.5 | **40.25** | +8.75 | 7 | 6 |
| GC AI | closed | 0 | 0 | 39.5 | 39.5 | — | 5 | 7 |
| Runlayer | closed | 0 | 0 | 34.5 | 34.5 | — | 6 | 8 |

\* Hologram's stack score is a manual override. Its posting body exceeded Clay's 8kb formula-access cap, so the formula engine read the cell as empty. All four tools (HubSpot, Clay, n8n, Zapier) were verified by hand against the posting's "What We're Looking For" section and the dimension hardcoded at 75. Correct value, uncomputed — see `day2_debrief.md` deviation #8.

### Arithmetic for the three rows that moved

```
Hyperbound  (75×35 + 10×25 + 100×20 + 100×20) / 100 = (2625 + 250 + 2000 + 2000) / 100 = 68.75
Hologram    (75×35 + 10×25 + 100×20 +  30×20) / 100 = (2625 + 250 + 2000 +  600) / 100 = 54.75
Suger.io    (25×35 + 30×25 +  60×20 +  60×20) / 100 = ( 875 + 750 + 1200 + 1200) / 100 = 40.25
```

The five closed postings scored 0 on stack in both passes, so their totals are unchanged from `day1_debrief.md` §5.

---

## What this cost and what it changed

**Cost:** 1.5 credits for all 8 rows.

**Rank churn:** 6 of 8 positions changed. The top of the list inverted — Turnkey fell from 1st to 3rd without its own score moving, and Hologram went from last to 2nd.

**The headline:** Hologram was ranked 8 of 8 on titles and 2 of 8 once the posting was read. Its posting named four of the exact tools in the ICP definition. Titles are a weak proxy for stack fit, and Day 1's ranking was measuring the ceiling imposed by its own input data, not the accounts.

**Stack-score distribution after re-scoring:** 75, 75, 25, and five zeros. Only the three live postings produced any stack signal at all — which means 62.5% of this dimension's coverage was lost to posting churn, not to the scoring logic.

---

## Threshold consequence

Step 2's contact waterfall was gated at `icp_score_v2 >= 50`, which selects **Hyperbound (68.75) and Hologram (54.75)** — the two rows the body scrape moved. The nearest row below is Turnkey at 44.5, a 10.25-point gap.

For the record, since this table makes it checkable: a threshold at ≥40 would have admitted five rows, not two — Turnkey (44.5), Lorikeet (42.5), Reducto (42.5) and Suger.io (40.25) alongside the two selected. `day2_debrief.md`'s threshold discussion names Suger.io as the marginal case because it was the only *live-posting* row in that band, and the argument there is about signal quality rather than score order. The three others in the band are all closed postings scoring on titles alone.
