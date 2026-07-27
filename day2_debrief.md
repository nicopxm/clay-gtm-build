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
| Fix A batch — 4 remaining Ashby rows (gc-ai, runlayer, reducto, turnkey) | 4 | 0.4 | 0.5 | 1,915.4 |
| **Total step spend** | | **1.4** | **1.5** | **1,915.4** |
| Step 2A — Surfe "Find people at company" (auto-run gated to `icp_score_v2 >= 50`) | 8 scanned → 2 executed | 0.2 | 0.2 | 1,915.2 |
| Step 2A escalation — Clay "Find Contacts at Company" (Hologram only, after Surfe miss) | 1 | 10.5 (displayed) / 0.5 (actual) | 0.5 | 1,914.7 |
| Step 2B — Icypeas "Find work email" (both rows) | 2 | 0.4 | 0.4 | 1,914.3 |
| **Step 2 total spend** | | **1.1** | **1.1** | **1,914.3** |

Small ledger variance (0.1 cr) on the Fix A batch — expected 0.4 cr for 4 rows, actual 0.5 cr. Suspected cause: Force Re-run flag captured Hyperbound alongside the 4 batch rows because Clay treated its already-scraped row as an eligible re-run target. Not worth further debugging — trivial dollar impact, but logged for the ledger integrity story: **when using Force Re-run in Clay, verify the exact row-set list, not just the count.**

**Ledger correction (applied 2026-07-26):** the Step 1 "Fix A batch" and "Total step spend" rows originally read 1,915.5, which contradicts their own arithmetic (1,915.9 − 0.5 actual = 1,915.4). Corrected to 1,915.4 to match the header and the table's own math. Step 2 opens from 1,915.4, not the 1,915.5 in earlier notes.

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

Ledger 1,915.4. Git commit: `Step 1: posting-body scrape + re-score (v2 stack + icp columns)`.

Ready for Step 2 (contact waterfall).

---

# Step 2: contact waterfall

**Date:** 2026-07-22
**Ledger opening balance (Step 2 open):** 1,915.4
**Ledger closing balance (Step 2 close):** 1,914.4
**Total step spend:** 1.2 credits
**Rows in scope:** 2 (Hyperbound, Hologram — both with icp_score_v2 >= 50)

## Threshold decision

Set contact-waterfall threshold at `icp_score_v2 >= 50`. Reasoning: the v2 distribution had a natural gap at 50 — Hyperbound (68.75) and Hologram (54.75) above, Turnkey (44.5) and everything else below. Threshold surfaced only rows where body-scrape enrichment confirmed real stack fit; excluded rows with dead postings (which had no meaningful signal beyond title-only fallback). Deliberately aggressive 75% cull rate. Alternative considered: threshold ≥40 would have added Suger (40.25) but Suger's live-body evidence explicitly de-emphasized its own Zapier mention — a hiring manager reading the write-up would question why we spent contact-lookup credits on a company whose posting told us the tool wasn't the core system.

## Waterfall structure — cheapest first, escalation only on miss

Three actions chained. Contact enrichment is inherently two-step (find person, then find email), so both steps got cheap-first waterfall structure.

### Step 2A — Find person at company

| Attempt | Provider | Rate | Rows run | Result |
|---|---|---|---|---|
| 1 (cheap) | Surfe "Find people at company" (SMB Coverage tag) | 0.1 cr | 2 (both) | Hyperbound HIT (Atul Raghunathan, Co-Founder & CTO/CRO). Hologram MISS. |
| 2 (escalation) | Clay "Find Contacts at Company" (Enterprise Accuracy/Coverage tags) | 0.5 cr | 1 (Hologram only) | HIT — Stephen Chin, Revenue Operations Manager, 7x Salesforce Certified |

Escalation gate: only ran on the specific row that missed. Clay Find Contacts auto-run formula was `{{company}} == "Hologram"`, which produced 1-row preview before running.

### Step 2B — Find work email for each surfaced person

| Provider | Rate | Rows run | Result |
|---|---|---|---|
| Icypeas "Find work email" | 0.2 cr | 2 | Hyperbound: atul@hyperbound.ai (`ultra_sure`, FOUND). Hologram: stephen.chin@hologram.io (`ultra_sure`, FOUND). |

Both emails came back at Icypeas's highest confidence tier on the first (cheapest) provider. No email-side escalation needed.

## What was built

Formula columns (all zero-credit):

- `contact_name` = `{{Name People}} || {{Fullname People}}` — coalesces Clay Find Contacts output (Hologram) with Surfe output (Hyperbound). OR operator treats empty strings as falsy in Clay's engine.
- `contact_linkedin` = same pattern, coalescing Clay's Url with Surfe's LinkedIn URL column
- (Fields differ per Clay's autogenerated column names — reference by autocomplete in the formula editor)

Auto-run gate formula reused across all three actions:

- Surfe: `{{icp_score_v2}} >= 50` — gated to 2 rows
- Clay Find Contacts: `{{company}} == "Hologram"` — gated to 1 row
- Icypeas: `{{contact_name}}.length > 0` (or equivalent) — gated to 2 rows

Total post-Step-2 populated data per row:

| Row | contact_name | contact_linkedin | title | email | certainty |
|---|---|---|---|---|---|
| Hyperbound | Atul Raghunathan | linkedin.com/in/atul-raghunathan | Co-Founder & CTO/CRO | atul@hyperbound.ai | ultra_sure |
| Hologram | Stephen Chin | linkedin.com/in/stephen-chin-9541819/ | Revenue Operations Manager | stephen.chin@hologram.io | ultra_sure |

## Deviations, learnings, and honest limitations

### 12. Clay cost display unreliable — action library vs. config panel vs. actual bill

Clay's action library showed 0.5 cr/row for "Find Contacts at Company" (By Clay). The action's config panel displayed 10.5 cr/row after fields were selected. Actual billed cost after run: 0.5 cr. Config-panel display was 20x higher than actual bill.

Zero credit impact this time (trivial spend), but on a higher-stakes action or a tight budget, working from the config-panel estimate would have blocked or triaged a run that was actually affordable.

**Lesson:** cost estimates from Clay's action config panel are not authoritative. The action library rate and the Usage panel post-run delta are. If the two disagree pre-run, ask Clay support or run a 1-row test to establish the real rate before scaling. Sprint-relevant version of this lesson: **for any Clay action where the config-panel cost differs from the action-library cost by >2x, verify with a 1-row test before batch-running.**

**Build-vs-buy angle:** n8n has no per-action cost display friction because it doesn't have per-action pricing at all — cost is API-call-count × external-API-pricing, deterministic and auditable. Clay's abstraction adds a real UX layer between what a builder plans to spend and what they actually spend.

### 13. Auto-run gate — verified working under real conditions

Surfe fired on 8 rows scanned but billed only 2 (Hyperbound + Hologram, the two meeting `{{icp_score_v2}} >= 50`). Clay's post-run Usage panel confirmed 0.2 cr delta, not 0.8 cr. **This is the mechanic that makes cost-ordered enrichment work.** Pre-run scanning is free; post-condition billing is what's actual.

**Design point:** the whole cost-discipline story rides on the auto-run gate actually gating. Verified.

### 14. Escalation cost story — cheap → mid, only on the row that mattered

The Surfe → Clay Find Contacts escalation was a 5x rate increase (0.1 → 0.5) but only fired on 1 of 8 rows. Effective cost of contact enrichment across the 2 in-scope rows: 0.2 (Surfe on both) + 0.5 (Clay escalation on 1) = 0.7 cr / 2 contacts = 0.35 cr avg per contact.

Blanket-running Clay Find Contacts on both rows would have cost 1.0 cr — **~40% more expensive** for the same result (Surfe was going to hit Hyperbound anyway; Clay was only needed for Hologram). Blanket-running Clay's built-in "Work Email" waterfall (1.1 cr on both rows) would have cost 2.2 cr — nearly 3x more than the manual chain.

**Lesson:** the cheap-first waterfall pattern saves credits precisely because miss rates aren't 100% — you spend the expensive provider only on the specific rows where the cheap provider missed. Blanket-running a waterfall abstracts this and undercuts the savings.

### 15. Founder-fallback matching — organic emergence, not a bug

Surfe returned Atul Raghunathan (Co-Founder & CTO/CRO) for Hyperbound. Title not in our specified list (Head of RevOps / VP Sales / etc.), but the seniority filter caught C-Level and the CTO/CRO dual-hat is exactly the "founder is running revenue" signal for YC-stage companies. Accepting the match.

**Design point:** this is the correct behavior for the pipeline to produce. Hyperbound hasn't hired a formal RevOps leader yet because they're 30 people with a founder still on ops. The right outbound target at this stage is the founder, not a nonexistent VP. The waterfall surfaced this without special-casing — the seniority filter's broad C-Level match caught it.

### 16. Surfe SMB coverage gap on Hologram → validated escalation path

Surfe's SMB Coverage tag misled on Hologram (~100-200 employees, mid-market boundary). Zero results returned. Clay Find Contacts (Enterprise tags) found 10 people matching the same broad criteria, top match being an exact-fit senior RevOps person.

This is a specific coverage-gap pattern worth naming: **provider tags describe indexing focus, not accuracy guarantees.** Surfe's SMB tag likely means "well-indexed for SMB" not "guaranteed hits on any SMB." Hologram sits at the boundary between SMB and mid-market, which is exactly where indexing gaps show up.

**Lesson:** don't blindly trust provider tags. The escalation path (cheap tag → broader coverage) exists precisely because tags are directional, not absolute.

### 17. Icypeas as email-tier winner — cheapest provider hit both rows at ultra_sure

Icypeas at 0.2 cr/row is the cheapest email provider in Clay's stack. Both rows returned `ultra_sure` certainty, `FOUND` status. No email-side escalation to LeadMagic (0.3), Enrow (0.2), Hunter (0.4), or Prospeo (0.5) needed.

**Lesson:** the cheap-first pattern applies to email lookup, not just people lookup. Icypeas isn't the accuracy leader on paper (no coverage tags), but empirically it hit on both tests. Escalation to higher-cost providers should be data-driven (Icypeas miss → try LeadMagic), not preemptive.

## Screenshot log — Step 2

- `14_surfe_config.png` — Surfe "Find people at company" config with `{{icp_score_v2}} >= 50` auto-run formula, title list, and 2-row preview visible
- `15_clay_find_contacts_hologram_config.png` — Clay "Find Contacts at Company" config with `{{company}} == "Hologram"` auto-run formula, title keywords, and 1-row preview visible
- `16_icypeas_config.png` — Icypeas "Find work email" config with `contact_name` mapped, auto-run formula, and 2-row preview visible
- `17_step2_final_table.png` — table view showing Hyperbound + Hologram rows with all Step 2 output columns populated (contact_name, contact_linkedin, title, email, certainty)

## Result

Two contacts, two ultra_sure work emails, ready for HubSpot push (Step 3):

| Company | Name | Title | Email | Certainty |
|---|---|---|---|---|
| Hyperbound | Atul Raghunathan | Co-Founder & CTO/CRO | atul@hyperbound.ai | ultra_sure |
| Hologram | Stephen Chin | Revenue Operations Manager | stephen.chin@hologram.io | ultra_sure |

Effective cost: 1.1 cr for 2 fully-enriched leads = 0.55 cr per lead. Compare to industry benchmarks in Day 3 write-up (ZoomInfo per-record cost, Apollo per-contact cost, etc.).

## Step 2 close

Ledger 1,914.3. Git commit: `Step 2: contact waterfall (Surfe → Clay escalation → Icypeas email lookup)`.

Ready for Step 3 (HubSpot push with deliberate two-push dedupe test).

---

# Step 3: HubSpot push
 
**Date:** 2026-07-27
**Ledger opening balance (Step 3 open):** 1,914.3
**Ledger closing balance (Step 3 close):** 1,914.3
**Total step spend:** 0 credits (HubSpot integration actions do not consume Clay enrichment credits)
**Rows in scope:** 2 (Hyperbound, Hologram — both with icp_score_v2 >= 50, both with `ultra_sure` email from Step 2)
 
## Credit ledger
 
| Action | Rows | Est cost | Actual cost | Running balance |
|---|---|---|---|---|
| Step 3 — Lookup object (Hyperbound, first push) | 1 | 0 | 0 | 1,914.3 |
| Step 3 — Create object (Atul Raghunathan, first push) | 1 | 0 | 0 | 1,914.3 |
| Step 3 — Re-run Lookup + Update object (Atul, second push, dedupe test) | 1 | 0 | 0 | 1,914.3 |
| Step 3 — Lookup + Create object (Stephen Chin, Hologram push) | 1 | 0 | 0 | 1,914.3 |
| **Step 3 total spend** | | **0** | **0** | **1,914.3** |
 
HubSpot integration actions in Clay do not consume enrichment credits — they operate against HubSpot's API directly. Every action config panel confirmed 0 credit cost before running. Ledger unchanged across all 4 Step 3 actions.
 
---
 
## Design decisions locked before push
 
### Dedupe pattern — Path 1 (explicit lookup-first)
 
Two viable dedupe patterns were considered:
 
- **Path 1 (chosen):** Chained Lookup → Update-or-Create actions with conditional gates. Clay's Lookup returns existing Contact ID if match found; auto-run gates route to Update (if ID present) or Create (if ID absent). Explicit lookup-first pattern makes the dedupe logic inspectable in the run history.
- **Path 2 (rejected):** Fire Create unconditionally and rely on HubSpot's native email-based dedupe to reject or silently update duplicates. Simpler but pushes the dedupe logic server-side into HubSpot's black box.
**Chose Path 1** because REALITY §3 (#24 CLOSED, 2026-07-21) established that the pipeline enforces 30-day domain-keyed dedupe explicitly, not via HubSpot's default behavior. Two-runtimes consistency required Clay to reproduce the explicit-check pattern, not delegate to HubSpot. Path 2 would have worked, but the write-up beat "same dedupe contract, two runtimes" requires an explicit gate to point at.
 
### Field mapping — 6 fields at Contact level
 
Per REALITY §3 the pipeline pushes `name, email, company, lead_source` and has the `icp_score` custom property created but never populated (writing to it ships in pipeline issue #29, not built). Clay's build writes to all 6 including the first-ever real values in `icp_score`.
 
Field-by-field mapping:
 
| HubSpot property | Clay column | Type |
|---|---|---|
| `firstname` | `{{first_name}}` (formula: split from contact_name) | Standard |
| `lastname` | `{{last_name}}` (formula: split from contact_name) | Standard |
| `email` | `{{email}}` (from Icypeas output) | Standard |
| `company` | `{{company}}` (seed data) | Standard |
| `lead_source` | `{{lead_source}}` (formula: constant "Clay GTM Table") | Custom |
| `icp_score` | `{{icp_score_v2}}` | Custom |
 
### Object-level scope — Contact only, not Contact + Company
 
HubSpot's data model has separate Contacts (people) and Companies (accounts) objects. Considered Path A (Contact-level push only) vs. Path B (Company + Contact with association). Chose Path A because REALITY §3 says the pipeline pushes as Contacts; two-runtimes consistency wins over data-model purism for this sprint. Real tradeoff: `icp_score` is a company-level signal on a person-level record. Noted for the write-up as an intentional call, not an oversight.
 
### Dedupe test protocol — deliberate two-push on Hyperbound
 
Sprint spec required: push one row, verify in HubSpot, push same row again, confirm update-not-duplicate, screenshot both states, then bulk push remaining rows. Executed as:
 
1. First push: filter Lookup + Create gates to `{{company}} == "Hyperbound"` only. Lookup returns "❌ No objects found." Create fires. Atul appears in HubSpot as new Contact ID `237877930071`.
2. Second push: force re-run Lookup on same row. Lookup returns Atul's Contact ID. Update fires against existing record. No new Contact created.
3. Verify HubSpot Contacts search for Atul returns exactly 1 entry. Dedupe verified.
4. Bulk push: widen gate to Hologram. Stephen Chin created as second Contact.
---
 
## What was built
 
Formula columns (all zero-credit):
 
- `first_name` = `{{contact_name}}.split(" ")[0]`
- `last_name` = `{{contact_name}}.split(" ").slice(1).join(" ")`
- `lead_source` = `"Clay GTM Table"` (constant string)
HubSpot integration actions (all zero-credit):
 
- **Lookup object** — searches HubSpot Contacts by email, populates `hubspot_contact_id` column with the Contact ID (as nested JSON object) or `"❌ No objects found"` on miss
- **Create object** — creates new Contact with 6-field mapping, auto-run gated to fire only when no existing Contact matches
- **Update object** — updates existing Contact by ID, auto-run gated to fire only when Lookup returned an existing ID
Auto-run gates across the chain:
 
- Lookup: `{{company}} == "Hyperbound"` (initial), then widened to `{{company}} == "Hologram"` for the Hologram push
- Create: `{{company}} == "Hyperbound" && {{hubspot_contact_id}} == "❌ No objects found"` (initial); simplified to `{{company}} == "Hologram"` for the Hologram push
- Update: nested-path check `{{hubspot_contact_id.results[0].id}}` — worked cleanly on Hyperbound's second push, unreliable on Hologram
---
 
## Deviations, learnings, and honest limitations
 
### 18. Clay Lookup returns status strings, not nulls
 
HubSpot Lookup action populates non-matching cells with the string `"❌ No objects found"` — a user-facing label with an emoji, not a null value. Downstream auto-run formulas checking for empty results via `!{{col}}` or `.length == 0` returned false against this content.
 
**Fix:** explicit string equality check against the exact "no results" string.
 
**Lesson:** Clay's action outputs are optimized for table-view readability, not for programmatic downstream use. Formula authors have to know the specific status strings each action emits, or test with a temp column before writing conditional gates.
 
### 19. Hologram push — simplified gate after Clay reference layer proved unreliable
 
The 3-action chain (Lookup → Update or Create) with nested reference paths (`{{col.results[0].id}}`) worked cleanly on Hyperbound but failed to resolve on Hologram's row after Lookup returned `"❌ No objects found"`. Multiple attempts at conditional gates evaluated false when they should have evaluated true. Rather than continue debugging Clay's reference resolution across chained action outputs mid-sprint, pushed Stephen with a simplified gate (`{{company}} == "Hologram"`) and relied on HubSpot's server-side email dedupe as the safety net.
 
**Real cost:** Hologram's push skipped the explicit lookup-first dedupe verification that Hyperbound got. Dedupe safety is preserved (HubSpot's native email dedupe would have caught any duplicate) but the "same contract, two runtimes" write-up beat is 1-of-2 clean.
 
**Lesson:** Clay's action-output references become brittle when chained across multiple HubSpot actions with nested response objects. The status strings and JSON wrappers Clay populates for readability compound with each chain step, making downstream conditional logic harder to write than it should be. For a production build, this would be worth investigating with Clay support or moving to a webhook-out pattern instead of Clay's built-in HubSpot actions. For this sprint, the simplified gate is the pragmatic call.
 
### 20. First-ever write to HubSpot's `icp_score` custom property
 
Per REALITY §3, the pipeline created the `icp_score` HubSpot custom property but never wrote to it — the scoring itself ships in pipeline issue #29 (not built as of this sprint). Clay's build wrote real values (68.75 for Hyperbound, 54.75 for Hologram) to that property for the first time. Both values verified via HubSpot's Contact record properties view.
 
**Not a deviation, a write-up beat worth logging alongside the deviations because it's the strongest single "same contract, two runtimes" moment in the sprint.** The pipeline team built `icp_score` as future-facing infrastructure targeting scoring pipeline issue #29. Clay's build closed the loop from "score column exists as target" to "score value populated as data" without the pipeline scoring being built yet. The two runtimes aren't just parallel — they're complementary within the same target schema. Clay's build proves the schema receives what the pipeline will eventually send.
 
### 21. Cache/refresh behavior on Lookup — sticky "not found" state
 
First re-run of Lookup after Atul's Create returned the same `"❌ No objects found"` result, despite Atul existing in HubSpot at that point. Toggle Auto-run OFF → ON on the Lookup config forced a fresh API call, which then returned Atul's Contact ID.
 
Working hypothesis: Clay caches Lookup results per-cell and requires an explicit state change to invalidate. Force Re-run flag or auto-run toggle change acts as the cache-bust.
 
**Lesson:** Clay's action re-run isn't always a fresh execution — it may return cached results. When testing dedupe or state-change behavior specifically, toggle the auto-run OFF then ON, or delete the cell value to force a fresh call. Worth naming in the debrief because this could bite anyone building a stateful workflow that depends on Clay observing HubSpot state changes mid-run.
 
### 22. Contact-level `icp_score` — intentional data-model tradeoff
 
Pushed `icp_score` to the Contact object, not to a Company object. The score is a company-level signal (based on firmographic + posting-body data about the account, not the person), but the pipeline convention per REALITY §3 is Contact-level pushes and two-runtimes consistency won over data-model purism.
 
**Design note for the write-up:** a production version would consider the score as a Company property with Contacts inheriting via association. Named as a real tradeoff so a hiring manager reviewing the debrief sees that the decision was made deliberately, not by omission.
 
---
 
## Screenshot log — Step 3
 
- `18a_hubspot_lookup_config.png` — Lookup action config with `{{company}} == "Hyperbound"` gate and Email input mapped to Icypeas output
- `18b_hubspot_create_config.png` — Create action config with 6 field mappings and status-string gate
- `19_hubspot_atul_created.png` — Atul's HubSpot Contact record with 6 field values visible (including `icp_score = 68.75` and `lead_source = "Clay GTM Table"` in All Properties view)
- `20_hubspot_update_config.png` — Update action config with nested-path Object ID and 6 field mappings
- `21_hubspot_dedupe_verified.png` — HubSpot Contacts list showing single Atul entry after two pushes (Create → Update)
- `22_hubspot_hologram_created.png` — Stephen Chin's HubSpot Contact record with 6 field values (Hologram push completed via simplified gate)
- `23_hubspot_both_contacts.png` — HubSpot Contacts list showing both Atul Raghunathan and Stephen Chin
---
 
## Result
 
Two contacts pushed to HubSpot, both with all 6 fields populated including the first-ever writes to the `icp_score` custom property.
 
| Company | Contact | Email | icp_score written | HubSpot Contact ID | Dedupe verified |
|---|---|---|---|---|---|
| Hyperbound | Atul Raghunathan | atul@hyperbound.ai | 68.75 | 237877930071 | Yes (Create → Update, single record confirmed) |
| Hologram | Stephen Chin | stephen.chin@hologram.io | 54.75 | populated (Contact ID assigned by HubSpot) | Partial (server-side native dedupe only; simplified gate bypassed explicit lookup-first check) |
 
## Step 3 close
 
Ledger 1,914.3 (no change — HubSpot integration actions do not consume Clay enrichment credits). Both target Contacts live in HubSpot with all 6 fields populated. Dedupe verified on Hyperbound via explicit lookup-first pattern (Create → Update, single record confirmed by HubSpot Contacts search). Hologram push completed via simplified gate; server-side email dedupe preserved the safety net.
 
Git commit: `Step 3: HubSpot push with dedupe test (Atul via lookup-first, Stephen via simplified gate)`.
 
Ready for Step 4 (Clay vs. n8n benchmark).
 