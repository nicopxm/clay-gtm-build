# day2_debrief.md — Step 1: posting-body scrape + re-score

**Date:** 2026-07-21
**Sprint day:** 2 of 3
**Step:** 1 of 4 (opener — body scrape + re-score)
**Ledger opening balance:** 1,916.9
**Ledger closing balance:** 1,915.4
**Total step spend:** 1.5 credits

---

## Credit ledger

| Action | Rows | Est cost | Actual cost | Running balance |
|---|---|---|---|---|
| Opening balance (Day 1 close) | — | — | — | 1,916.9 |
| First scrape: 8 gate-passers, Clay Scrape Website, JS ON, no delay | 8 | 0.8 | 0.8 | 1,916.1 |
| Fix A test — Lorikeet (Ashby, JS + 5s delay) | 1 | 0.1 | 0.1 | 1,916.0 |
| Fix A test — Hyperbound (Ashby, JS + 5s delay) | 1 | 0.1 | 0.1 | 1,915.9 |
| Fix A batch — 4 remaining Ashby rows (gc-ai, runlayer, reducto, turnkey) | 4 | 0.4 | 0.5 | 1,915.5 |
| **Total step spend** | | **1.4** | **1.5** | **1,915.5** |

Small ledger variance (0.1 cr) on the Fix A batch — expected 0.4 cr for 4 rows, actual 0.5 cr. Suspected cause: Force Re-run flag captured Hyperbound alongside the 4 batch rows because Clay treated its already-scraped row as an eligible re-run target. Not worth further debugging — trivial dollar impact, but logged for the ledger integrity story: **when using Force Re-run in Clay, verify the exact row-set list, not just the count.**

---

## What was built

Three new formula columns and one enrichment action:

1. **Enrichment:** Clay `Scrape Website by Clay` action against `careers_or_posting_url`, JavaScript Rendering ON, Scrape Delay 5s (added after first pass revealed Ashby SPA rendering was incomplete without it). Auto-run gated to `{{icp_gate}} == true`. Output column: `Bodytext` (child of `posting_body_scraped` object wrapper).
2. **Formula columns:**
 - `test_body_length` = `LEN({{Bodytext}})` — helper column, plain number output.
 - `posting_body_usable` = `IF({{company}} == "Hologram", true, {{test_body_length}} >= 500)` — boolean, with Hologram-specific override due to 8kb formula-access cap.
 - `body_text_for_scoring` = `LOWER({{posting_title}} & " " & IF({{posting_body_usable}}, {{Bodytext}}, ""))` — lowercased search string per row. Live rows: title + body. Dead rows: title only.
 - `score_stack_v2` = manual Hologram override at 75; otherwise `.includes()` match against `body_text_for_scoring` on `hubspot`, `clay`, `n8n`/`zapier`, `outreach,`/`outreach.`/`outreach and`/`apollo,`/`apollo.`/`apollo and` (25 pts each, cap 100).
 - `icp_score_v2` = `({{score_stack_v2}}*35 + {{score_hiring}}*25 + {{score_size}}*20 + {{score_funding}}*20) / 100` — same weights as Day 1's `icp_score`, only stack reference swapped.

---

## Deviations, learnings, and honest limitations

### 1. Auto-run formula syntax — string vs boolean gotcha

First attempt used `{{icp_gate}} == "pass"` on the assumption `icp_gate` was a string column. Clay's condition-check UI caught the type mismatch pre-run and refused to fire. Zero credits spent on the failed attempt. Corrected to `{{icp_gate}} == true` (or bare `{{icp_gate}}`, equivalent for boolean fields). Row-count preview then confirmed 8 rows, ran cleanly.

**Lesson:** Clay's pre-run condition-check validates types and blocks silent misfires. n8n's formula surfaces don't offer the same protection — you'd burn the API call to discover the same bug. Worth flagging in the build-vs-buy write-up as a real Clay advantage.

### 2. Ashby SPA rendering — Clay's default JS toggle wasn't enough

Six of eight gate-passer URLs were Ashby (`jobs.ashbyhq.com`). Clay's default JS render toggle (no delay) returned only the SPA fallback string (`"You need to enable JavaScript to run this app."`) with no posting content. Adding `Scrape Delay = 5s` gave Ashby's SPA time to hydrate, and subsequent scrapes returned real posting bodies.

**Lesson:** JS rendering is not binary. When scraping SPAs (Ashby, Ember apps, some Vue/React deploys), the scraper needs a delay for hydration, not just JS execution. Clay exposes this as a config field — worth flagging as one of the small usability wins over more opaque enrichment tools that don't let you tune scrape timing per action.

### 3. Data freshness — 5 of 8 postings closed between seed and enrichment

62.5% of the gate-passer URLs returned Ashby's `"Job not found"` state in the scrape output. Postings had closed in the ~7 days between Day 1 seed collection and Day 2 enrichment. Preserved these 5 rows in the output rather than dropping them, marked their `posting_body_usable` as `false`, and let `score_stack_v2` fall back to title-only matching (which scored 0 across all 5, same as Day 1's title-only baseline).

**Lesson (design):** This is why you scrape body content after the free gate, not before. Cost-ordered enrichment means we spent 1.4 credits to discover a real data-freshness gap. If we had paid-enriched all 50 rows upfront, the same churn rate implies ~31 dead postings would have been enriched at full cost — 62.5% waste. **Design point for the write-up: waterfall ordering isn't only about per-row cost, it's about not paying for stale data.**

### 4. Formula bug caught pre-run — SPA fallback as a "dead posting" signal

First draft of `posting_body_usable` used the string `"you need to enable javascript"` as a signal for "posting unusable." Ashby renders that string in every page's HTML shell, live or dead. Would have falsely marked 3 live rows as unusable and dropped them from body-based scoring. Caught by inspecting body content before writing the detection logic.

**Lesson:** When scraping SPA pages, verify the failure signal against a known-good *live* sample before writing detection logic. The SPA-fallback text is a static shell string, not a state indicator.

### 5. Formula bug caught pre-run — `"Job not found"` as a signal, second attempt

Second draft used `"Job not found"` as the dead-posting signal. Ashby may render this string in its SPA state bundle regardless of live/dead status (confirmed via Ctrl+F check on Hyperbound's live body: string not present, but the risk was real enough to switch signals). Moved to a length-based check: `test_body_length >= 500`. Robust to ATS-specific error text and generalizes across scrapers.

**Lesson pattern:** **When a substring signal keeps failing across attempts, switch to a shape signal (length, structure, presence of any content at all).** Substring detection is fragile against templated error pages that share vocabulary with real content. Length-based detection is dumb but robust.

### 6. Clay formula engine — nested LEN comparison bug

`LEN({{col}}) >= 500` returned false on every row despite `LEN({{col}})` alone returning correct numbers. Working hypothesis: Clay's formula engine has a type-coercion issue when comparing function return values to numeric literals inline. Fix: decompose into two columns — `test_body_length` returning raw LEN, `posting_body_usable` returning `{{test_body_length}} >= 500`.

**Lesson:** When Clay's formula engine misbehaves, decompose into intermediate columns. Same principle as breaking a complex SQL query into CTEs. Kept the length column in the table anyway — it's inspection-useful.

### 7. Clay column reference — nested object vs. child column

Scrape action created `posting_body_scraped` as an object wrapper and `Bodytext` as the child column containing the plain string body. Referencing `{{posting_body_scraped}}` in a concat expression returned the literal string `[object Object]` because it referenced the wrapper, not the string leaf. `LEN({{posting_body_scraped}})` worked by accident (measured the serialized form). Fix: reference `{{Bodytext}}` everywhere.

**Lesson:** Clay's scrape actions produce a column tree, not a single column. Formula references should target the leaf-level string column, not the parent. Autocomplete in the formula editor shows the full tree — worth checking every reference against it before saving.

### 8. Clay platform limitation — 8kb formula-access cap

Cells storing >8kb of scraped content are accessible in Clay's detail view but return null/undefined to the formula engine (LEN returns 0, `.includes()` returns false regardless of actual content). Hologram's posting body hit this cap. Options considered:

- **(a)** Re-scrape with character limit (0.1 cr, uncertain signal preservation).
- **(b)** Hardcode score based on manual content verification (0 credits, requires isolated override).

**Chose (b) as a one-row exception.** Manual verification against Hologram's posting: HubSpot ✓, Clay ✓, n8n ✓, Zapier ✓ — all four appear in the "What We're Looking For" section. Score locked at 75 via `IF({{company}} == "Hologram", 75, ...)` override in `score_stack_v2`. `posting_body_usable` also carries the override so downstream logic passes Hologram through.

**Design point for the write-up:** if this build scaled beyond a portfolio project, the fix would be an upstream scrape config that returns markdown-summarized bodies under 8kb — this is on the Clay platform side, not addressable by formula changes. Filed as a real platform limitation.

### 9. Clay AI formula assistant — hallucinated regex direction

Clay's AI generated a formula that reversed the substring test: `/{{col}}/i.test("hubspot")` uses the column value as a regex pattern to match against the literal term `"hubspot"`, not the other way around. Every test returned false, every scoring branch collapsed to 0, and only Hologram's manual override survived. The formula ran without errors — compile-clean but semantically inverted. Manually corrected to standard `.includes()` form.

**Lesson (build-vs-buy):** AI-generated formulas in Clay require the same review as any generated code. Compile-clean is not correctness. The formula ran and returned bad data silently. n8n's formula surfaces are typically standard JS expressions where a similar bug would look obviously wrong in review; Clay's AI generator abstracts syntax at the cost of introducing errors that pass syntactic checks. **Write-up beat: AI helpers reduce syntax friction but can introduce silent semantic errors. The mitigation is human review — same as any other generated code.**

### 10. Score_stack_v2 — Outreach/Apollo false positive, tightened by punctuation boundary

First working pass with `.includes("outreach")` and `.includes("apollo")` produced Hyperbound = 100 with all 4 categories hit. Manual review of Hyperbound's body found the "outreach" match was triggering on the common noun ("routing, and outreach into one system") not the Outreach.io tool. Same class of risk existed for "apollo" (mission/god/finance firm collisions).

Considered ".io" suffix requirement — rejected because it would miss legitimate brand references without the domain suffix.

Went with punctuation-boundary matching: `outreach,` / `outreach.` / `outreach and` / same variants for apollo. Trade: catches most real tool references while filtering Hyperbound's specific false positive. Verified against Hyperbound's body — none of the tool-reference patterns present, so Hyperbound correctly drops to 75 (HubSpot + Clay + Zapier). HubSpot, Clay, n8n, Zapier retained as bare-term matches — brand names unique enough in tech-hiring text not to collide.

**Lesson:** Deterministic keyword scoring makes anomalies inspectable. The false positive was caught by eyeballing one row's body against its score in about 30 seconds, and fixed in another 30. LLM-based sentiment scoring would fix false positives at the cost of per-row credits and non-determinism. **The design payoff of "no LLM in the scoring loop" is that anomalies are cheap to find and cheap to fix.**

### 11. Suger's Zapier de-emphasis — known scoring limitation, not patched

Suger's posting explicitly de-emphasizes Zapier ("we care far more about your ability to build practical systems than whether you know how to configure zapier workflows. we still use tools like zapier when useful, but they are not the core system"). The substring match counts the Zapier mention as +25 pts anyway. Real cost: Suger's score_stack_v2 is 25 when a fair reading might put it closer to 0-10.

**Chose not to patch this.** It's the same class of issue as the Outreach false positive but harder to detect syntactically. Documented in the write-up as: **"the tool sees term presence, not intent."** Real limitation of deterministic keyword scoring. LLM-based sentiment scoring would fix it but adds cost and non-determinism.

---

## Screenshot log

- `06-top8.png` (Day 1) — 8 rows filtered by `icp_gate = pass`
- `07a_scrape_config_inputs.png` — scrape action input config (URL field, source column)
- `07b_scrape_config_runsettings.png` — scrape action run settings (JS Rendering ON, Scrape Delay 5s)
- `08_scrape_run_complete.png` — 8 rows with Bodytext populated after first scrape run
- `09_ashby_dead_posting.png` — dead Ashby posting cell detail showing "Job not found"
- `10_posting_body_usable.png` — table view with `test_body_length` and `posting_body_usable` columns side by side (3 true, 5 false)
- `11_icp_score_v2.png` — table sorted by `icp_score_v2` desc, both v1 `icp_score` and v2 columns visible

---

## Result

Before/after top-8 table in `outputs/step1_before_after_top8.md`. Headline finding: **Hologram moved from rank 8 to rank 2 (+26.25 icp points) after body-scrape enrichment.** The lead Day 1's title-only scoring ranked worst was one of the top 2 fits once the posting was actually read.

## Step 1 close

Ledger 1,915.5. Git commit: `Step 1: posting-body scrape + re-score (v2 stack + icp columns)`.

Ready for Step 2 (contact waterfall).
