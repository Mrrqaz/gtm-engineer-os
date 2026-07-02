---
name: exec-revenue-briefing
description: Turns raw pipeline and revenue data into a story-spine executive or board briefing (headline, context, signal, implication, action) built on a fixed bookings/coverage/win-rate/NRR/CAC-payback KPI stack. Two output modes, a full quarterly board version and a lighter weekly exec version. Enforces a no-orphaned-bad-news rule (every red metric needs a named owner and next action before it reaches the exec) and a no-surprises rule (any metric that flips from green to red without prior warning gets flagged as its own issue). Run before a board meeting, before a weekly exec sync, or whenever asked to "write the board update," "prep the revenue briefing," or "summarize the quarter for Alex."
user_invocable: true
---

# Exec Revenue Briefing

Adapted from gtmagents' `gtm-agents` repo (github.com/gtmagents/gtm-agents), specifically `plugins/revenue-analytics/skills/exec-briefing-kit/SKILL.md` and `revenue-health-dashboard/SKILL.md`. The story-spine narrative format and the bookings/coverage/win-rate/NRR/CAC-payback KPI stack are theirs; the CEO-fit rules below (no orphaned bad news, no surprises) and the Rivergate Data worked example are new for this repo.

This skill does not analyze pipeline from scratch. It's the layer that turns an already-analyzed pipeline into a briefing a CEO can act on in two minutes. It consumes `pipeline-forecast-review`'s weekly output as one of its inputs (see Intended Integrations) — this skill does not re-derive stage probabilities or forecast math, it packages what forecast-review already found.

## Operating Rules (read first)

- **Story spine, not a data dump.** Every metric that appears gets run through the same five-part structure: headline, context, signal, implication, action. A number with no "so what" attached doesn't belong in the briefing.
- **The KPI stack is fixed.** Bookings, pipeline coverage ratio, win rate, NRR, CAC/payback period. Don't add a metric because it's interesting this week, and don't drop one because it's flat. A flat metric still gets a one-line mention — "on plan, no action needed" is a valid, useful line.
- **No orphaned bad news.** Any metric flagged red or at-risk MUST have a named owner and a stated next action before it's allowed into the final output. If a red flag has no owner yet, that is itself the action item — "needs an owner assigned" is the interim next action, never silence.
- **No surprises, ever.** Before finalizing, diff this cycle's flags against last cycle's. Anything red this time that was green last time with no prior warning is a meta-issue in its own right (see Step 5). Alex's standing complaint is being surprised on the board call by a number the team already knew was slipping — this rule exists specifically to stop that.
- **Mode changes depth, not honesty.** The weekly exec version is shorter because Alex already has quarterly context, not because bad news gets softened. A red flag is a red flag in either mode.
- **Never invent a number.** If a KPI input is missing, say "not reported this cycle" and treat the gap itself as a flag if it's a KPI that's normally reported. Don't interpolate or estimate to fill a blank cell.

## Intended Integrations (not live in this demo)

- **Data warehouse / BI layer** (e.g. Snowflake + a semantic layer, or a tool like Hex/Mode) would be the source of truth for the raw bookings, pipeline, win-rate, NRR, and CAC inputs. In production this skill would query pre-aggregated views, not raw CRM rows, so the KPI math is consistent run over run.
- **`pipeline-forecast-review` (upstream skill)** would hand this skill its weekly stage-probability-checked forecast and any flagged forecast/reality mismatches. This skill treats that output as a trusted input for the pipeline-coverage and win-rate lines, not something to re-derive.
- **Google Slides API / Google Docs API** would generate the actual board deck or exec doc from the story-spine content, so the narrative structure below maps onto real slides/sections rather than staying as chat output.
- None of these are wired in this repo. Inputs come from whatever CRM export, pasted numbers, or prior skill output the user provides in-session. If an input is missing, the relevant KPI line says "not reported this cycle" — see Operating Rules above.

## The KPI Stack

Every briefing, either mode, reports on these five metrics if the data exists. Each one gets a status: **on plan**, **watch**, or **red**.

| Metric | What it answers | Watch/red trigger (illustrative, tune per company) |
|---|---|---|
| **Bookings** | Are we closing what we said we'd close, this period vs. plan? | Watch: <90% of period target. Red: <75% or two consecutive periods below plan. |
| **Pipeline coverage ratio** | Is there enough open pipeline in the current period to plausibly hit the number? (open pipeline ÷ remaining target) | Watch: below 3x. Red: below 2x with less than 6 weeks left in the period. |
| **Win rate** | Of what we're closing, how much are we actually winning vs. losing/no-decision? | Watch: down >5pts vs. trailing quarter. Red: down >10pts, or below a stated floor for the segment. |
| **NRR (net revenue retention)** | Is the existing book expanding or shrinking net of churn/downgrade? | Watch: below 100% (net contraction). Red: below 90%, or a sequential drop of >5pts. |
| **CAC / payback period** | Is new revenue getting more or less expensive to acquire, and how fast does it pay back? | Watch: payback stretching beyond the company's stated target window. Red: payback beyond target by more than 50%, or CAC up >20% quarter over quarter with no bookings-mix explanation. |

Status thresholds are illustrative starting points — the real numbers should come from the company's own plan and board-agreed guardrails (Alex's phrase: "only escalates when a number moves outside guardrails"). Don't invent guardrails; if none are stated, use the illustrative ones above and say so.

## Step 1: Gather Inputs

Pull, in order of preference:

1. `pipeline-forecast-review`'s most recent weekly output, if it exists in this session or was pasted in — this is the primary source for pipeline coverage, win rate, and any stage-probability flags.
2. Any CRM export, KPI dashboard, or numbers the user pastes in directly for bookings, NRR, or CAC/payback (these typically don't come from pipeline-forecast-review).
3. The prior cycle's briefing, if one exists, for the diff needed in Step 5.

If an input is missing for a KPI that's normally tracked, mark it "not reported this cycle" in Step 2 rather than skipping it silently — a metric going dark is itself worth Alex knowing about.

## Step 2: Score Each KPI

For each of the five KPI stack metrics, assign on plan / watch / red using the triggers above (or company-stated guardrails if provided). Write one factual line per metric: the number, the comparison point (vs. plan, vs. trailing period), and the status.

## Step 3: Build the Story Spine for Each Flagged Metric

For every metric that is **watch** or **red**, and for anything else worth surfacing (a bookings mix shift, a single deal driving the whole miss), write it through this exact five-part structure. Skip parts for metrics that are cleanly on-plan — those get one line in Step 2 and nothing more.

1. **Headline** — the one-sentence version, plain language, no jargon. What happened.
2. **Context** — the number(s) and the comparison that make the headline true. Plan vs. actual, trailing-period comparison, or segment breakdown.
3. **Signal** — why this is happening, based on what's actually known (a specific deal, a rep, a segment, a stage-probability mismatch pipeline-forecast-review caught). Don't speculate beyond what the source data supports; say "cause not yet isolated" if that's the truth.
4. **Implication** — what this means for the number Alex actually cares about (will we hit the quarter, is the book shrinking, is CAC becoming unsustainable). Connect it up, don't leave it as an isolated fact.
5. **Action** — the named owner and the specific next step, with a date if one exists. This is the mandatory field under the no-orphaned-bad-news rule — see Step 4.

## Step 4: Enforce No Orphaned Bad News

Before anything gets written to final output, check every watch/red item from Step 3 has a non-empty **owner** and **action** field.

- If a red-flagged metric has no clear owner yet, the action becomes "owner not yet assigned — flagging for Alex to assign" and that gap is itself listed as part of the finding, not hidden.
- Never let a red or watch line reach the final briefing with a blank action field. This is a hard gate, not a style preference — it's Alex's explicit standing rule (see `context/people/alex-novak-ceo.md`).

## Step 5: No-Surprises Check

Compare this cycle's KPI statuses against the prior cycle's (from the last briefing, if one exists in this session or was provided).

- If a metric was **on plan** or **watch** last cycle and is **red** this cycle, check whether last cycle's briefing (or `pipeline-forecast-review`'s output in between) contained any warning signal for it. If there was a warning and it just wasn't escalated loudly enough, that's a process gap, name it.
- If there was genuinely no prior warning, i.e., the metric looked clean last cycle and flipped hard this cycle, flag **the surprise itself** as a separate line item in the briefing, distinct from the metric's own story spine. State plainly: this moved further/faster than the data available last cycle would have predicted, and note what would need to change (a leading indicator, an earlier check) to catch it next time.
- This is the rule this skill exists to enforce. Alex's interaction log (2026-06-20) records exactly this failure mode: a forecast narrative that didn't match what showed up on the board deck, root-caused to a stage-probability mismatch that a proper forecast-discipline check would have caught upstream. The no-surprises check is the downstream backstop — if `pipeline-forecast-review` already caught something and this skill still lets it land as a surprise in the briefing, that's a defect in this skill, not an acceptable outcome.

## Step 6: Assemble the Output — Choose Mode

**Full quarterly board version** — used for the actual board meeting. Comprehensive:

```
[Company] Revenue Briefing — Board — [Quarter/Date]

HEADLINE
[one paragraph: the state of the business in KPI-stack terms, so-what first]

KPI STACK
Bookings:            [number vs plan] — [status]
Pipeline coverage:   [ratio] — [status]
Win rate:            [% vs trailing] — [status]
NRR:                 [% ] — [status]
CAC / payback:       [figure vs target] — [status]

STORY SPINE — [metric name] ([status])
Headline:     ...
Context:      ...
Signal:       ...
Implication:  ...
Action:       [owner] — [next step, date if known]

[repeat Story Spine block per flagged metric]

NO-SURPRISES CHECK
[any metric that flipped without warning, named as its own issue, or
"no unflagged reversals this cycle"]

APPENDIX
[segment/rep-level detail if provided, for board members who want to dig in]
```

**Weekly exec version** — used for the standing exec sync. Headline plus deltas only, assumes quarterly context is already known:

```
[Company] Revenue Briefing — Weekly — [Week of Date]

HEADLINE
[one to two sentences, so-what first]

DELTAS SINCE LAST WEEK
Bookings:            [delta] — [status if changed, else "unchanged"]
Pipeline coverage:   [delta] — [status if changed, else "unchanged"]
Win rate:            [delta] — [status if changed, else "unchanged"]
NRR:                 [delta] — [status if changed, else "unchanged"]
CAC / payback:       [delta] — [status if changed, else "unchanged"]

FLAGGED THIS WEEK
[Story Spine, compressed to headline + action, for anything watch/red]

NO-SURPRISES CHECK
[one line: clean, or names the metric that flipped without warning]
```

Default to the weekly version unless the user asks for the board/quarterly version, or the date context makes clear a board meeting is imminent.

## Worked Example (Rivergate Data, fictional, weekly mode)

Say pipeline-forecast-review's latest output flagged that two deals in the $180K late-stage bucket were carried at 80% stage probability but had no scheduled next step in three weeks, and that win rate for the SMB segment dropped 12 points this week. Bookings and NRR came in flat vs. last week. No CAC figure was provided this cycle.

```
Rivergate Data Revenue Briefing — Weekly — Week of 2026-06-27

HEADLINE
Pipeline coverage looks fine on paper but is inflated by two stalled
late-stage deals; SMB win rate dropped hard this week. Bookings and NRR
are flat, nothing new there.

DELTAS SINCE LAST WEEK
Bookings:            unchanged — on plan
Pipeline coverage:   flat headline, but see flag below — watch
Win rate:            SMB segment down 12pts — red
NRR:                 unchanged — on plan
CAC / payback:       not reported this cycle

FLAGGED THIS WEEK

Pipeline coverage (watch)
Headline: Two $180K late-stage deals are carried at 80% probability with
no next step booked in three weeks — coverage math is overstating what's
actually live.
Action: Dana (VP Sales) to re-stage or confirm next-step dates on both
deals by Friday; forecast-review flagged the same gap upstream.

SMB win rate (red)
Headline: SMB win rate dropped 12 points this week, no single deal
explains it.
Context: 4 of 9 closed-lost this week were SMB, vs. typical 1-2.
Signal: cause not yet isolated — competitive loss reasons not yet
logged consistently for 3 of the 4.
Implication: if this holds another week it starts pulling the quarter
number down, not just a noisy week.
Action: Dana to pull loss-reason detail on all 4 by next Monday; owner
confirmed, no prior escalation needed yet since this is week one of the
drop.

NO-SURPRISES CHECK
Clean this week. The pipeline-coverage issue was already flagged by
pipeline-forecast-review before it reached this briefing, which is the
system working as intended, not a surprise. SMB win rate is a first
occurrence, not a reversal of a prior "on plan" call, so it's a new red
flag, not a no-surprises violation.
```

## Rules

- Every metric in the KPI stack appears in every briefing, even if the line is just "on plan, no action."
- No watch/red line ever reaches final output without a named owner and a next action. If there's no owner, "assign an owner" is the action — never a blank field.
- Run the no-surprises diff every cycle, not just when something looks obviously wrong. The check is only useful if it's automatic.
- If a metric flips green-to-red with no trace of prior warning in the record, name that as its own line, separate from the metric's own story spine — it's a process finding, not just a number update.
- Never pad the weekly version with quarterly-level detail Alex already has. If nothing changed, say "unchanged" and move on.
- Never soften a red flag because the mode is "lighter." Mode controls length, not honesty.
