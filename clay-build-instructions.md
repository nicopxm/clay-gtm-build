# Clay Build — Day 1 Instructions

> **Context for Claude Code:** These are step-by-step build instructions for a Clay (clay.com) table build on a free trial. Build in this exact order. Screenshot after every completed step — free trial means every screen gets captured the first time.
>
> **Setup task:** Create a `/screenshots` folder now. Name files in build order: `01-import.png`, `02-waterfall.png`, etc. Future-you assembling the write-up will thank you.
>
> **Caveat:** These instructions can't see Clay's UI, and it changes often. What to build and what to configure is specified; exact button names may differ — **adapt, don't hunt for exact wording.**

---

## Step 1 — Workspace + import (15 min)

1. New table → import `seed_list.csv`.
2. **Verify:** 110 rows, 6 columns mapped correctly.
   - Watch that `posting_date` and `stage_hint` come in as **text**, not auto-parsed dates on mostly-empty columns.
3. Screenshot the imported table (`01-import.png`).
4. **Do nothing else yet** — no enrichments on import.

---

## Step 2 — Firmographic waterfall (the credit-discipline centerpiece)

Three fields are needed for the gate and scoring:

- Employee count
- HQ / geo
- Funding stage + last raise date

Build the waterfall in **cost order**:

1. **First column(s):** Clay's free/cached company data. Clay has built-in company enrichment from domain that costs little or nothing — use that tier first.
2. **Then cheap provider columns** for the same fields, each with a conditional run: **only run if the free column came back empty.** This is the "skip paid if free filled it" rule — in Clay it's the "Only run if" condition on the enrichment column, something like `free_employee_count is empty`.
3. **Funding data (stage + raise date)** is the G4 field — set it up, but note it should effectively run **last**; this is enforced via the gate structure in Step 3.

**Expected failures:** The 9 blank-domain rows will fail enrichment — that's expected. Leave them; they'll die at the gate and join the honest-numbers count.

Screenshot the waterfall columns **with their conditional-run settings visible** (`02-waterfall.png`) — that conditional config is the design decision, capture it.

---

## Step 3 — The gate (one formula column)

Create a boolean column `icp_gate` as a conjunctive AND of:

| Gate | Condition |
|------|-----------|
| **G1** | `posting_title` matches `RevOps\|Revenue Operations\|GTM Engineer\|Sales Op(s\|erations)\|GTM (Strategy & )?Operations` (case-insensitive). Clay formula columns take regex, or use its AI-assist formula builder. **Verify against the two "GTM Strategy & Operations" rows — they must now pass.** |
| **G2** | Employee count ≥ 20 AND ≤ 200 — **from the free-tier column only.** Empty = FAIL. No fallback reference to the paid column. |
| **G3** | US-based / US-focused, from seed or free data. Empty = FAIL. |
| **G4** | Funding stage is Series A or B. |

**Structural note on G4's "runs last":** The clean way in Clay is to make the funding enrichment column itself conditional on G1 AND G2 AND G3 passing (via a pre-gate formula column, or the conditions inline), then:

```
icp_gate = pre_gate AND G4
```

That way the cheap provider is never called for rows already dead on free checks — the cost-ordered-gate story working literally.

Screenshot the gate formula (`03-gate.png`).

**Record the first real number:** how many of 110 passed — and the breakdown of how many failed on **missing data** vs. failed on **criteria**. That's the write-up's headline stat.

---

## Step 4 — AI column `gtm_motion` (conditional)

- One AI/Claygent column, **Only run if `icp_gate = true`**.
- Inputs: homepage + pricing text (Clay can scrape, or feed the domain).
- **Paste the locked prompt from the Project verbatim.**
- Spot-check 5 outputs for valid JSON and sane classifications.
- Screenshot one open row showing the JSON (`04-gtm-motion.png`).

---

## Step 5 — Scoring columns (5 formula columns, zero credits)

Four dimension columns, each 0–100, straight from the locked table:

| Column | Logic |
|--------|-------|
| `score_stack` | +25 per match of HubSpot / Clay / n8n-or-Zapier / Outreach-or-Apollo in posting text (and fingerprint if available), cap 100 |
| `score_hiring` | Seniority tier: +50 Head/Director/VP, +30 Manager, +10 other. +30 if "(multiple postings)" in title. +20 if posting <30 days. Cap 100. **With `posting_date` blank on 109 rows, the +20 simply doesn't fire — note it, don't fake it.** |
| `score_size` | 100 if 40–120, 60 if 20–39 or 121–200 |
| `score_funding` | 100 if raise <12mo, 60 if 12–24mo, 30 if older/unknown date |

Then the composite:

```
icp_score = (score_stack*35 + score_hiring*25 + score_size*20 + score_funding*20) / 100
```

**Formula columns only** — the "no AI in scoring, four readable cells" line depends on it.

Screenshot a scored row (`05-scoring.png`).

---

## Step 6 — Stop line

1. Sort by `icp_score`.
2. Screenshot the top 10 (`06-top10.png`).
3. Sanity-check them: *"Would I actually want to contact these?"*
4. **Then stop.**

Contact waterfall and HubSpot push are **Day 2 work** — and more importantly, the push should happen fresh, with the dedupe test done deliberately, not tacked on at the end of a long build session.

If steps 1–5 are finished today, you're a half-day ahead. Bank it.
