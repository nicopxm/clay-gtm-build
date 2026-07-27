# REALITY.md — Ground Truth: Lead Intelligence Pipeline State
**Dated: 2026-07-14** · **Authority: this file OUTRANKS PROJECT_BRIEF.md wherever they conflict.**

Purpose: the Clay GTM Table project references Wop's Lead Intelligence Pipeline for its benchmark and HubSpot mapping. This file states what actually exists in the pipeline repo as of the date above, so no chat in this Project claims unbuilt features as shipped. If a write-up draft asserts something not listed as EXISTS here, flag it and stop.

---

## 1. Enrichment workflow — EXISTS (with caveats)
**Real, described in PIPELINE_REFERENCE.md (uploaded to this Project):**
- Enrichment mechanics: scrape / tech-stack fingerprint / news RSS sub-workflows
- Key DECISIONS rationale (no-headless-browser, fingerprinting, RSS, robots.txt posture)
- HubSpot mapping and ICP scoring schema

**Not finished — do not present as a complete flow:**
- Orchestration (#23): not wired as of this date

**Write-up rule:** describe enrichment as "core scraping + signals workflows built; news and orchestration in progress." The DECISIONS entries may be used qualitatively in the build-vs-buy section (constraints engineered manually that Clay abstracts).

## 2. Cost instrumentation — DOES NOT EXIST
- There is NO per-lead cost data. Cost-per-lead logging is Sprint 3 work (it instruments the Claude API call, which is not built).
- What exists: a target (<$0.02/lead) and a plan. A target is not a measurement.
- Latency logging arrives with #23 (possibly imminent, not confirmed).

**Benchmark rule (amends PROJECT_BRIEF.md section 4):** the n8n side of the comparison is limited to match rate, setup effort, and qualitative constraints. Any n8n cost figure must be an ESTIMATE (API pricing × observed call counts) and labeled as such. No benchmark table may present n8n cost or latency as measured data unless #23 has shipped AND produced real numbers by Day 2 PM. Do not hold the sprint for it.

## 3. HubSpot mapping — PARTIAL
**Real today:**
- Contacts pushed with: name, email, company, lead_source
- `icp_score` property created in HubSpot — but never written to
- Dedupe: 30-day domain-keyed update-not-insert, ENFORCED pipeline-side as of #24 (2026-07-21). Same email + different domain = new lead (job-change case handled); within 30 days = update in place, no duplicate contact.

**Not built (Sprint 4):**
- Pushing score, summary, and draft email (depends on #29 scoring, not yet built)

**Clay push rule:** target only the fields real today (name/email/company/lead_source) plus writing `icp_score` — which the Clay build does for the first time. Mirror the pipeline's dedupe logic (domain-keyed, 30-day update-not-insert) so both systems land leads the same shape — that consistency is itself a design decision worth one write-up line. Present the mapping as "current vs. planned," clearly split.

## 4. ICP config — DEPLOYED (SCORING NOT YET WIRED)
- `docs/ICP-CONFIG.md` EXISTS: defines dimensions, weights-sum-100 rule, deterministic aggregation.
- `configs/icp.default.json` EXISTS and is runtime-verified as of #28 (2026-07-21). FlowSignal ICP, ≤200-employee scope, 5 weighted dimensions summing to 100 (`config_version 1.0.0-v1`). Valid config loads clean; all 4 invalid cases (bad JSON, weights≠100, misordered thresholds, missing file) fail loud and alert via Resend.
- **NOT built: the scoring itself.** The Claude API call that reads this config and produces an `icp_score` per lead is #29 — not yet built. **No lead has a score yet.** The config loads and validates at runtime, but nothing consumes it to score.

**Scoring rule:** the Clay scoring formula implements the same schema contract (weights sum to 100, deterministic — model scores dimensions, code computes total). Talking point: "same scoring contract, two runtimes." Do NOT claim the pipeline produces scores today — it loads and validates the config that scoring will use; scoring ships in #29.

---

## Standing rules for this Project
1. Anything labeled DOES NOT EXIST or NOT built here may only appear in deliverables as explicitly planned/estimated/in-progress — never as shipped.
2. If pipeline status changes mid-sprint (#22, #23, #24 landing), Wop updates this file with a new dated entry below. Chat claims of "it shipped" without an entry here don't count.
3. Honest limitations go IN the write-up, not around it. "Instrumentation lands in Sprint 3; here's the estimate meanwhile" is the desired register.

## Change log
- 2026-07-14: Initial version. Source: Wop's repo audit correcting the assumed upload list.
- 2026-07-15: #22 SHIPPED — news enrichment built (Google News RSS, quoted-name query, top 5 items / 90 days; zero results = success; write-failure alerting verified via Resend send log). Enrichment trio (scrape, fingerprint, news) complete as sub-workflows. #23 orchestration remains NOT wired — leads still do not get enriched automatically end-to-end. Section 2 unchanged: no cost data; latency logging still pending #23.
- 2026-07-17: Clay build phase COMPLETE (ran 2026-07-15→17, 2 calendar days vs. planned 1). 50-row table live in Clay trial (110-row list capped by free tier; deliberate subset, logic in subset_notes.md). Waterfall → gate → Claygent → scoring all built; 8/50 passed icp_gate; 88.1 of 2,005 trial credits spent. Full record: day1_debrief.md in clay-gtm-build repo (private). Known limitation: score_stack = 0 on all rows (titles-only seed data); fix queued Day 2.
- 2026-07-17: NOT YET DONE from Clay sprint: posting-body scrape/re-score, contact waterfall, HubSpot push (dedupe untested — REALITY §3 #24 still not enforced), benchmark run. Benchmark note stands: n8n cost figures remain estimates, not measured. Clay trial expiry: 2026-07-29 — hard deadline for Day 2 + benchmark.
- 2026-07-20: #26 FIXED — tech-stack detection restored, re-verified through production path; benchmark asterisk retired.
- 2026-07-21: #24 CLOSED — 30-day domain-keyed dedupe (update-not-insert) enforced pipeline-side; §3 "Designed, NOT enforced" now stale. #28 CLOSED — ICP config (configs/icp.default.json: FlowSignal ICP, ≤200-employee scope, 5 weighted dimensions summing to 100) deployed AND runtime-verified: valid config loads clean (config_version 1.0.0-v1), all 4 invalid cases (bad JSON, weights≠100, misordered thresholds, missing file) fail loud + alert via Resend. §4 "SCHEMA ONLY" is now FALSE — the ICP config loads and validates at runtime. Scoring itself (the Claude call, #29) NOT yet built — no lead has an icp_score yet.
