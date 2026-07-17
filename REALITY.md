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

**Designed, NOT enforced:**
- Dedupe: 30-day update-not-insert (ARCHITECTURE decision #6). Enforcement is issue #24, scheduled next.

**Not built (Sprint 4):**
- Pushing score, summary, and draft email

**Clay push rule:** target only the fields real today (name/email/company/lead_source) plus writing `icp_score` — which the Clay build does for the first time. Present the mapping as "current vs. planned," clearly split.

## 4. ICP config — SCHEMA ONLY
- docs/ICP-CONFIG.md EXISTS: defines dimensions, weights-sum-100 rule, deterministic aggregation.
- `configs/icp.default.json` DOES NOT EXIST (Sprint 3).

**Scoring rule:** the Clay scoring formula implements the same schema contract (weights sum to 100, deterministic). Talking point: "same scoring contract, two runtimes." Do not claim a config file exists.

---

## Standing rules for this Project
1. Anything labeled DOES NOT EXIST or NOT built here may only appear in deliverables as explicitly planned/estimated/in-progress — never as shipped.
2. If pipeline status changes mid-sprint (#22, #23, #24 landing), Wop updates this file with a new dated entry below. Chat claims of "it shipped" without an entry here don't count.
3. Honest limitations go IN the write-up, not around it. "Instrumentation lands in Sprint 3; here's the estimate meanwhile" is the desired register.

## Change log
- 2026-07-14: Initial version. Source: Wop's repo audit correcting the assumed upload list.
- 2026-07-15: #22 SHIPPED — news enrichment built (Google News RSS, quoted-name query, top 5 items / 90 days; zero results = success; write-failure alerting verified via Resend send log). Enrichment trio (scrape, fingerprint, news) complete as sub-workflows. #23 orchestration remains NOT wired — leads still do not get enriched automatically end-to-end. Section 2 unchanged: no cost data; latency logging still pending #23.
