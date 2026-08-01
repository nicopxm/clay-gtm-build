# clay-gtm-build

**A cost-ordered GTM enrichment system built in Clay — 50 companies in, 8 qualified, 2 CRM-ready contacts in HubSpot — then benchmarked against the n8n pipeline I already run in production.**

Manual account research is slow, and the databases that replace it run $15k+/year on seat-based pricing. This repo is the full build record of a system that does the same job on usage-based cost: gate cheaply, enrich only what survives, score deterministically, and land verified contacts in a CRM with dedupe that actually holds.

Every credit spent is ledgered. Every deviation is numbered. The parts that didn't work are in here too.

📄 **[Read the full write-up →](writeup.md)**

---

## Results

| | |
|---|---|
| **Funnel** | 50 companies → 8 passed the ICP gate → 2 contacts pushed to HubSpot |
| **Disqualified** | 42 (33 on criteria, 9 on missing data — by policy, with zero rescue lookups) |
| **Cost** | 90.7 Clay credits total · 11.3 cr/account fully loaded · 45.4 cr per CRM-ready lead |
| **Contact accuracy** | 2/2 work emails at the provider's highest confidence tier |
| **Timeline** | Table live in 2 calendar days; enrichment → contact → CRM → benchmark over the following two weeks |
| **Benchmark** | Same 8 accounts through both systems, 5 comparison dimensions, verdict on when to pick which |

---

## The system

```
seed list (company, domain, posting URL, title)
    ↓
firmographic waterfall     Clay Enrich 0.5 cr/row → Harmonic 4 cr/row (conditional)
    ↓
ICP GATE                   title regex → headcount → geo → funding stage
                           cost-ordered · conjunctive AND · missing free data = fail
    ↓  8 of 50 survive
posting-body scrape        0.1 cr/row · runs after the gate, never before
    ↓
scoring                    4 formula columns, weights sum to 100, zero credits, no LLM
    ↓  threshold ≥ 50
contact waterfall          Surfe 0.1 cr → escalate to Clay 0.5 cr only on miss → Icypeas email
    ↓
HubSpot                    explicit lookup → update-or-create · dedupe verified
```

The design principle in one line: **every expensive step sits behind a cheap one that can veto it.**

---

## What the build actually surfaced

- **Column order was worth 160 credits.** Putting the 4 cr/row funding provider behind three free checks meant it ran on 10 rows instead of 50. That single ordering decision saved more than everything else in the build cost combined.
- **The worst-ranked lead was one of the two best.** Title-only scoring put Hologram 8th of 8. After scraping the actual job posting — which named HubSpot, Clay, n8n, and Zapier in the requirements — it moved to 2nd. Titles are a weak signal for stack fit.
- **62.5% of the qualifying job postings closed within a week** of being collected. Because enrichment ran *after* the gate, that churn cost 0.1 cr/row to discover instead of full price across the whole list.
- **An AI-generated formula ran clean and returned garbage.** Clay's formula assistant inverted a regex test — compile-valid, semantically backwards, every row silently scoring zero. Caught by review, not by an error. Same category as the n8n aggregation node that would have merged 8 companies' data under one lead ID if it had been run in batch: caught by reading the code, not by observing the corruption.
- **n8n's news enrichment reported 3 of 8 companies as clean successes while returning 100% irrelevant articles.** Name collisions — a government initiative, a stock-listed optics company, and generic business copy. A status vocabulary with no way to say "succeeded and worthless."

---

## Repo contents

| File | What's in it |
|---|---|
| **[writeup.md](writeup.md)** | The full narrative — problem, design decisions, economics, results, build-vs-buy verdict, limitations. **Start here.** |
| [day1_debrief.md](day1_debrief.md) | Table architecture, credit ledger, gate results with per-row failure attribution, scoring findings |
| [day2_debrief.md](day2_debrief.md) | Steps 1–4: body scrape and re-score, contact waterfall, HubSpot push, benchmark. 27 numbered deviations with root cause and fix |
| [PROJECT_BRIEF.md](PROJECT_BRIEF.md) | The spec, written before the build — ICP definition, scoring weights, sprint plan, out-of-scope list |
| [REALITY.md](REALITY.md) | Ground-truth state of the incumbent n8n pipeline, written before the sprint so no claim in the write-up could outrun what's actually shipped |
| [outputs/step1_before_after_top8.md](outputs/step1_before_after_top8.md) | Title-only vs. posting-body scoring, all 8 accounts, arithmetic shown |
| [outputs/step4_comparison_table.md](outputs/step4_comparison_table.md) | Clay vs. n8n across 5 dimensions with the verdict |
| [outputs/step4_benchmark_capture.csv](outputs/step4_benchmark_capture.csv) | All 24 n8n sub-workflow runs — status, counts, timing, per-row notes |
| [subset_notes.md](subset_notes.md) | Why the 110-row seed list became 50, and how proportions were preserved |
| [claygent_prompt.md](claygent_prompt.md) | The one AI classification prompt used in the build |
| [clay-build-instructions.md](clay-build-instructions.md) | The Day 1 build script, written before touching Clay. Left unedited — it specifies 110 rows and a `06-top10.png` screenshot, neither of which survived contact with the free-tier row cap |
| [seed_list.csv](seed_list.csv) / [seed_list_50.csv](seed_list_50.csv) | The full 110-row input list and the 50-row subset actually built on |
| [screenshots/](screenshots/) | 26 screenshots, from starting credit balance through both contacts live in HubSpot |

**If you have five minutes:** [writeup.md](writeup.md) — Design Decisions and Economics.
**If you want to check my work:** [day2_debrief.md](day2_debrief.md) — the 27 deviations, including the ones I chose not to fix and why.

---

## Stack

Clay (enrichment waterfall, Claygent, formula columns, HubSpot actions) · Harmonic · Surfe · Icypeas · HubSpot · n8n + Supabase (the incumbent system being benchmarked)

---

## Honest limitations

Documented in full at the end of [writeup.md](writeup.md), summarized here because they matter for reading the numbers:

- **Deterministic scoring reads term presence, not intent.** One company's posting actively de-emphasized a tool it mentioned; the keyword match scored it anyway. Named rather than patched — the fix is the LLM-in-the-scoring-loop the design deliberately excludes.
- **n=2 on the contact waterfall.** The escalation economics are real arithmetic on the rows that ran. Two contacts demonstrate the pattern works; they don't establish provider hit rates.
- **Dedupe verified 1 of 2 cleanly.** The explicit lookup-first check worked on the first contact; the second went through a simplified gate after Clay's chained-action references wouldn't resolve. No duplicate was created — HubSpot's server-side dedupe held — but that row didn't get the same verification.
- **One score is a manual override.** Hologram's posting body exceeded Clay's 8kb formula-access cap. The four tools were verified by hand and the dimension hardcoded. Correct value, uncomputed.
- **50 rows, not 110.** Clay's free tier caps tables at 50. The subset was built to preserve category proportions and both edge-case title variants.

---

*Built by [Nico](https://github.com/nicopxm). The target list is companies hiring for RevOps and GTM Engineering roles — which makes the output of this project my own outbound list. That was intentional.*
