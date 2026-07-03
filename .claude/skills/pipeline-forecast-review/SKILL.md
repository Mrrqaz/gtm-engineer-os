---
name: pipeline-forecast-review
description: Weekly pipeline/forecast review. Tiers every open deal into Commit/Best-Case/Pipeline/Upside against explicit exit criteria, checks coverage and accuracy against numeric KPI guardrails, then fetches each rep's open deals and scores them on momentum, stakeholder coverage, and risk signals to produce a prioritized action list, not a static report. Flags any deal that drops a tier week-over-week and requires a written reason. Run weekly ahead of the Friday forecast call, or on request for a mid-week gut-check.
user_invocable: true
---

# Pipeline Forecast Review

Forecasting on gut feel is how a VP Sales gets surprised by slippage in front of the board. This skill replaces "I feel good about it" with a tier a deal has to earn, a coverage ratio that has to hold, and a scored, prioritized list of what to act on before Friday, not a dump of every open deal.

**Adapted from:** the 4-tier forecast structure, KPI guardrails, and weekly cadence are adapted from gtmagents' `gtm-agents` repo (github.com/gtmagents/gtm-agents), `plugins/sales-pipeline/skills/forecast-discipline/SKILL.md`. The fetch→score→prioritize mechanic (momentum, stakeholder coverage, risk-signal scoring) is adapted from amplemarket's `skills` repo (github.com/amplemarket/skills), `skills/ae-pipeline-review/SKILL.md`. Neither is copied verbatim; both are re-implemented here against Rivergate Data's stack and Dana Whitfield's stated preferences.

## Operating Rules (read first)

- **A tier is earned by exit criteria, not by a rep's confidence.** If a deal doesn't meet a tier's stated criteria, it doesn't sit in that tier, regardless of how the rep describes it in the CRM notes.
- **Never re-tier a deal without a stated reason.** Every tier change, up or down, gets one line of "why." A downgrade requires it before the deal-desk sync, not after.
- **This is an action list, not a status report.** The output ranks what needs a decision this week. A healthy deal with no risk signal doesn't need a line of prose just to prove it was checked.
- **Guardrail breaches are flagged, not silently absorbed into the total.** If coverage or accuracy misses threshold, say so explicitly at the top of the output, don't bury it in a table.
- **Reps update their own deals first.** This skill does not overwrite a rep's stage or close-date fields. It reads what's there, tiers it against criteria, and flags mismatches for the rep or manager to resolve.
- **No live CRM integration in this demo.** Every fetch step runs on data the user pastes in or a CRM export. If nothing is provided, say so and stop, don't fabricate deal rows (see Intended Integrations).

## The Four Tiers

Every open deal gets exactly one tier this week. A deal qualifies for a tier only if it meets **all** of that tier's criteria.

| Tier | Exit criteria (must meet all) |
|---|---|
| **Commit** | Verbal yes from economic buyer + signed paperwork in legal/procurement review + close date falls inside this period |
| **Best-Case** | Strong, multi-threaded champion confirmed + budget confirmed by buyer (not assumed by rep) + no verbal yes yet |
| **Pipeline** | Qualified opportunity, active engagement in the last 14 days, champion identified but not yet multi-threaded or budget-confirmed |
| **Upside** | Early-stage or speculative: first meeting held or reply received, no confirmed champion, no confirmed budget, close date is a guess |

A deal that meets none of the Pipeline-tier minimums (no active engagement in 14 days, no identified champion) doesn't get a tier. It gets flagged as a risk signal (see Step 3) and excluded from the forecast roll-up until it's requalified or disqualified.

## KPI Guardrails (numeric, non-negotiable)

Check these every run and report a pass/fail, not just a number:

- **Pipeline coverage ratio ≥ 4x new-logo target.** Total Pipeline + Best-Case + Commit value this period must be at least 4x the period's new-logo revenue target. Below 4x is a coverage-gap flag for the exec forecast call, not something to quietly note.
- **Commit-tier forecast accuracy ≥ 90%.** Measured against what actually closed last period: of everything called Commit at the start of the period, at least 90% must have closed by period end. Below 90% means the Commit bar was applied too loosely and needs tightening before this week's calls are finalized.
- **Slip rate ≤ 10% quarter-over-quarter.** Deals pushed from this period to next period, as a share of total deals in the period-start forecast, must stay at or under 10%. Above that is a pattern, not noise, and gets called out by name (which deals, which reps).

## Weekly Cadence

1. **Manager prep (reps update their own deals).** Before this skill runs, reps update stage, close date, and next-step notes on their own open deals. This skill reads that state, it doesn't originate it.
2. **Deal-desk sync.** Run Steps 1–4 below. Flag anomalies: a deal marked Commit with no signed paperwork, a stage that doesn't match the tier it's claiming, a stage probability manually set out of line with the deal's actual stage (the exact mismatch that surfaces on the board deck if it isn't caught here), a close date that's slipped without a stage change. This sync is where mismatches get resolved before numbers go up the chain.
3. **Exec forecast call.** Roll up the reconciled numbers into Commit and Best-Case totals for leadership. Only tier totals and guardrail pass/fail go into this call. The rep-level detail stays at the deal-desk layer.

## Step 1: Fetch Open Deals

Pull every rep's open-deal list for the period: deal name, stage, amount, close date, last-activity date, and any notes on champion/stakeholder status. In this demo, this comes from whatever CRM export or pasted deal list the user provides. If nothing is provided, say so and stop; don't invent deal rows (see Intended Integrations).

## Step 2: Tier Each Deal

Apply the exit criteria table above to every deal. Record the tier, and for any deal that doesn't clear even Pipeline-tier minimums, mark it "unqualified, risk flag" instead of forcing it into a tier it hasn't earned.

**Compare against last week's tier for the same deal.** Any deal that moved to a *lower* tier than last week (Commit → Best-Case, Best-Case → Pipeline, Pipeline → Upside, or dropped to unqualified) is a **downgrade escalation**. It requires one written reason before it's included in this week's output: a stalled champion, a budget freeze, a stakeholder change, whatever actually happened. No reason, no downgrade acceptance; flag it back to the rep first.

## Step 3: Score Each Deal

For every deal still in a tier (Commit through Upside), score it on three axes:

- **Momentum**: recency of activity. Active in the last 7 days = high; 8–13 days = medium; 14+ days = **stalled** (this alone is also a standalone risk signal, not just a momentum score).
- **Stakeholder coverage**: multi-threaded (2+ confirmed contacts, ideally spanning champion + economic buyer) vs. single-threaded (one contact only). Single-threaded on anything above Pipeline tier is a risk signal on its own: one person leaving kills the deal.
- **Risk signals**: explicit flags, not a vibe. Stalled 14+ days (no activity), competitor mentioned in notes/call transcript, champion went quiet (no response across 2+ touches), or champion changed jobs/title (LinkedIn or email bounce signal).

A deal with high momentum, multi-threaded coverage, and zero risk signals needs no further line in the output beyond its tier total. A deal with any risk signal gets surfaced.

## Step 4: Prioritize and Output

Don't output a flat table of every deal. Rank by what needs a decision this week:

1. **Downgrade escalations** (Step 2): always first, always need a reason attached or a follow-up ask.
2. **Commit/Best-Case deals with an active risk signal**: highest dollar impact on the number leadership sees Friday.
3. **Guardrail breaches**: coverage below 4x, Commit accuracy below 90%, or slip rate above 10%, called out with the specific deals or reps driving it.
4. **Pipeline/Upside deals with a risk signal**: lower dollar impact, but worth a mention if the pattern is repeating (e.g. the same rep has three stalled deals).

Deals with no risk signal and no tier change get compressed into the tier-total line, not itemized.

## Worked Example (Rivergate Data, fictional)

Dana Whitfield adopted this exact tiering on 2026-06-16 (`context/decisions-log.md`) because her AE team was calling forecasts on gut feel and the board kept getting surprised by slippage. She wants the tier and coverage ratio before any narrative, and she's skeptical of "AI will fix pipeline" framing, so the output leads with numbers, not a pitch.

Say this week's export shows 14 open deals across 3 reps, with last week's tiers available for comparison.

```
Pipeline Forecast Review - week of 2026-06-29

GUARDRAILS
- Coverage: 3.1x new-logo target (FAIL, needs 4x). Driven by thin Best-Case
  tier; only 2 deals sit there this week.
- Commit accuracy (trailing period): 92% (PASS, 11 of 12 called deals closed).
- Slip rate: 14% QoQ (FAIL, 2 of 14 deals pushed to next period, above the
  10% ceiling). Both slipped deals belong to the same rep, Jordan.

DOWNGRADE ESCALATIONS (require written reason before roll-up)
1. Northwind Logistics, Best-Case to Pipeline. Reason logged by rep: economic
   buyer went on leave, champion can't confirm budget until she's back
   (no date given). Accepted, flagged for re-check next week.
2. Halden Group, Commit to Best-Case. No reason logged yet. HOLD, kicked
   back to rep before this deal counts in Friday's Commit total.

COMMIT/BEST-CASE DEALS WITH ACTIVE RISK
1. Ferro Systems (Commit, $42k): stalled 16 days, single-threaded on the
   champion only. Champion hasn't responded to last 2 touches. This is the
   largest deal in Commit and it's showing two risk signals at once. It needs
   a direct escalation call this week, not another email.
2. Kestrel Health (Best-Case, $28k): competitor (Salesloft) mentioned in
   the last call transcript. Multi-threaded, momentum still active. Lower
   urgency than Ferro but worth a competitive-positioning follow-up.

PATTERN FLAG
Jordan has 2 of the 2 slipped deals this period, both stalled 14+ days
before slipping. Worth a 1:1 on pipeline hygiene, not just this week's
numbers.

TIER TOTALS (for Friday's exec call)
Commit: $118k (incl. Ferro, pending Halden resolution)
Best-Case: $61k
Pipeline: $204k
Upside: $89k (informational only, not rolled into coverage math)

FOCUS THIS WEEK
Resolve Halden's missing downgrade reason before Friday, it's currently
inflating the Commit number. Then get Ferro back on track, it's carrying
the coverage-gap risk on its own.
```

## Intended Integrations (not live in this demo)

- **HubSpot/Salesforce reporting API** would power Step 1, pulling every rep's open-deal list, stage, amount, close date, and last-activity timestamp directly instead of waiting for a pasted export. It would also carry the actual stage field, so Step 2's stage-vs-tier mismatch check could run automatically.
- **A data warehouse (e.g. Snowflake/BigQuery via a reverse-ETL layer)** would power the trailing-period accuracy check in the KPI guardrails, pulling what was actually called Commit at period-start and matching it against what genuinely closed, instead of relying on a manually reconstructed comparison.
- Neither is wired into this repo. This is a demonstration of the tiering, guardrail, and scoring logic, not a live deployment. Every fetch step here runs on pasted or exported data; if none is given, the skill says so and stops rather than inventing deal rows.

## Rules

- A tier is never assigned on rep confidence alone. It's earned against the stated exit criteria, full stop.
- Every downgrade needs a written reason before it's accepted into the roll-up. No reason means the deal goes back to the rep, not into Friday's number.
- Guardrail breaches are named explicitly, with the deals or reps driving them, not folded quietly into a total.
- The output is a ranked action list. A clean deal with no risk signal and no tier change gets a line in the tier total, not a paragraph.
- Never fabricate a deal, a close date, an activity timestamp, or a stakeholder detail. Missing data is a "no export provided" stop, not a gap to fill in.
- Rep-level detail stays at the deal-desk sync. Only tier totals and guardrail pass/fail go to the exec forecast call.
