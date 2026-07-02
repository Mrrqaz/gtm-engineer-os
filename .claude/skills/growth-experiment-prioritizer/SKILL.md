---
name: growth-experiment-prioritizer
description: Scores and ranks growth experiments and acquisition-channel bets on a shared scale so competing ideas can be compared instead of argued about. Routes each proposed idea through a 4-question intake to RICE, ICE, or a dedicated channel-investment evaluation (CAC/LTV/payback), then outputs a ranked backlog, not a single answer. Run when someone proposes a new experiment, campaign, or channel bet, when the backlog needs re-ranking after new data, or when asked "should we invest in X channel."
user_invocable: true
---

# Growth Experiment Prioritizer

The job isn't to bless one idea, it's to force every proposed experiment and every channel bet onto the same comparable scale so the backlog reflects actual expected value, not whoever pitched loudest last. Two different questions get asked here, and they need two different tools: "which experiment should we build next" (RICE/ICE) and "should we keep spending on this channel" (unit economics). Confusing them produces bad calls in both directions, a channel doing 3:1 LTV:CAC scored on RICE looks unremarkable next to a flashy small test, and a scrappy no-budget experiment forced through a CAC model returns garbage because it has no unit economics yet.

**Adapted from:** deanpeters' `Product-Manager-Skills` repo (github.com/deanpeters/Product-Manager-Skills) — `skills/prioritization-advisor/SKILL.md` (RICE/ICE/Value-Effort/Kano routing engine) and `skills/acquisition-channel-advisor/SKILL.md` (4-step channel evaluation with CAC/LTV/payback thresholds). This skill narrows that broader PM toolkit to the two frameworks that matter for GTM experimentation and adapts the worked example to Rivergate Data.

## Operating Rules (read first)

- **Output is always a ranked list, never a single verdict.** The point of a shared scale is comparison. If only one idea is in front of you, still score it and say what it would need to beat to make the current backlog.
- **Route before scoring.** Don't default to RICE because it's the most familiar. Run the Step 1 intake every time, a channel-investment question dressed up as an experiment idea is the most common misroute.
- **Never invent Reach, Confidence, CAC, or LTV numbers.** If the requester doesn't have a number, ask for their best estimate and mark it `(estimated)`, or ask for the input. A backlog full of confident-looking guesses is worse than one that's honest about what's unknown.
- **Confidence scores are a discipline problem, not a math problem.** The most common way this framework gets gamed is inflating Confidence to justify a favorite idea. When a Confidence input looks unsupported by any stated evidence (no prior test, no comparable channel data, no user research), flag it and ask what it's based on before using it.
- **Score on the same time horizon.** Reach and Impact must be normalized to the same period (e.g., per month) across every item being compared, or the ranking is meaningless.
- **A channel evaluation is not an experiment score.** Never force channel-investment questions through RICE/ICE. They get the 4-step unit-economics workflow in Step 3 instead.

## Intended Integrations (not live in this demo)

- **A shared backlog tool (Linear or Notion)** would hold the live ranked backlog as a real board, so Marcus's team sees the same ranking Sales and Product see, instead of a scoring doc that only exists in one person's head or a Slack thread.
- **A data warehouse (e.g. Snowflake or BigQuery, via a dbt-modeled marketing mart)** would supply the real CAC, LTV, and payback inputs for Step 3 directly from billing and ad-spend data, instead of requiring someone to hand-type them in.
- **HubSpot/Salesforce** would supply Reach inputs (accounts/contacts in a given segment) for RICE scoring, so Reach isn't an eyeballed guess.
- None of these are wired into this repo. This is a demonstration of the scoring and routing logic; every number in the worked example below is either seeded in `context/` or explicitly marked as an estimate supplied at run time.

## Step 1: Intake — Route the Idea

Ask (or infer from what's already been provided) these four questions before scoring anything:

1. **Is this a proposal to build/ship something (a feature, a campaign, a landing page, a new sequence), or a proposal to keep/start/stop spending on an existing acquisition channel?**
   - Build/ship → go to Step 2 (RICE or ICE).
   - Channel spend decision → go to Step 3 (channel evaluation). Stop here, do not also RICE-score it.
2. **How big is the bet?** (only relevant if routed to Step 2)
   - Cross-functional, multi-week, needs real Reach/Effort estimates → **RICE**.
   - Small, single-owner, ship-in-days-to-a-couple-weeks test → **ICE**.
   - When genuinely unsure, default to ICE for the first pass, upgrade to RICE only if it survives and needs a real resourcing conversation.
3. **Does anyone have a confidence number, or is this a pure gut call?**
   - If there's no supporting evidence at all (no prior data, no comparable), that's fine, but the Confidence score should land low (≤30%) and the skill should say why.
4. **What would "worked" mean, and by when?** If nobody can answer this, the idea isn't ready to score yet, it's ready for a 10-minute scoping conversation first. Say so instead of forcing a number.

## Step 2A: RICE Scoring

`RICE Score = (Reach × Impact × Confidence) / Effort`

| Variable | Definition | Scale |
|---|---|---|
| **Reach** | Number of people/accounts affected in the scoring period | Raw count, e.g. "420 trial signups/month" or "180 target accounts/quarter" |
| **Impact** | How much it moves the target metric per person/account reached | 3 = massive, 2 = high, 1 = medium, 0.5 = low, 0.25 = minimal |
| **Confidence** | How sure the team is about the Reach and Impact estimates | 100% = high (real data), 80% = medium, 50% = low (mostly guessing) |
| **Effort** | Total person-weeks across all functions to ship it | Raw number, e.g. 2 = two person-weeks |

**Named pitfall:** RICE systematically overweights Reach, so a niche-but-high-value bet (say, an enterprise-tier expansion play touching 15 accounts but worth 40% of ARR) loses to a low-value idea that merely touches more people. When a candidate has small Reach but high per-account value, either note that RICE is undercounting it and rank it up manually with a stated reason, or run it through the channel-evaluation lens on value per unit instead of raw Reach.

## Step 2B: ICE Scoring

`ICE Score = Impact × Confidence × Ease`

Faster than RICE, useful for small tests that don't need a Reach estimate to be worth greenlighting.

| Variable | Definition | Scale |
|---|---|---|
| **Impact** | Expected effect on the target metric if it works | 1–10 |
| **Confidence** | How sure the team is this will work | 1–10 (map from %: 9–10 = ~90%+, 5–6 = ~50-60%, 1–3 = ~10-30%) |
| **Ease** | How simple it is to implement (inverse of effort) | 1–10, 10 = trivial, 1 = very hard |

**Named pitfall:** ICE is fast precisely because it's coarse, which makes it the easier framework to game. A 1–10 Confidence slider invites optimistic rounding ("I feel good about this, call it an 8") with no Reach math to check it against. Treat any ICE Confidence above 7 with no stated evidence the same way Operating Rules flags it: ask what it's based on before trusting the ranking.

## Step 3: Acquisition-Channel Evaluation (separate workflow)

Use this instead of RICE/ICE whenever Step 1 routes to a channel spend decision (scale this channel, keep testing it, kill it, or evaluate a brand-new one).

**1. Unit Economics Check**
- **CAC** (Customer Acquisition Cost) = total fully-loaded channel spend ÷ customers acquired in the period.
- **LTV:CAC ratio** — illustrative thresholds: **below 3:1 is concerning** (channel likely losing money once churn and servicing cost are counted), **3:1 to 5:1 is healthy**, **above 5:1** is strong but worth checking whether the channel is under-invested relative to its return.
- **Payback Period** (months to recover CAC from gross margin) — illustrative thresholds: **under 12 months is healthy** for most B2B SaaS motions, **12–18 months is workable** if the business has the cash runway, **over 18 months is a red flag** unless there's a clear strategic reason (land-and-expand, category creation).

**2. Customer Quality Check**
- Compare the channel's cohort against company-average benchmarks on: 90-day retention, expansion rate, and NPS/support-ticket volume if available.
- A channel with acceptable CAC but customers who churn faster or expand slower than average is quietly worse than the CAC number suggests, flag it explicitly rather than letting the CAC number stand alone.

**3. Scalability Check**
- **Magic Number** = net new ARR in a quarter ÷ sales & marketing spend in the *prior* quarter. Illustrative thresholds: **above 0.75 = efficient, worth scaling**, **0.5–0.75 = workable, keep testing**, **below 0.5 = inefficient at current spend**.
- Ask: is there headroom in this channel's addressable audience, or does spend efficiency degrade sharply past current levels (diminishing returns, audience saturation)? A channel can pass economics and quality checks and still be near its ceiling.

**4. Recommendation**
Land on exactly one of four calls, and state the number(s) that drove it:
- **Scale** — LTV:CAC ≥ 3:1, payback ≤ 12–18mo, quality checks clean, Magic Number ≥ 0.75, and real headroom. Increase spend.
- **Test** — economics are borderline or data is thin (new channel, small sample). Keep current spend, run a bounded test with a pre-declared threshold before committing more.
- **Kill** — LTV:CAC below 3:1 *and* no credible path to improve it (quality check also weak, or payback over 18mo with no strategic offset). Stop spend, redirect budget.
- **Invest-to-Learn** — economics are currently bad or unproven, but there's a strategic reason to keep a small, capped spend running (new-category bet, competitive-intel value, land a logo type Sales needs). Name the cap and the review date explicitly, this is not a blank check.

## Step 4: Output — Ranked Backlog

Never return a single score in isolation. Present every item scored in this session (or pulled from the standing backlog if re-ranking) as one table, sorted highest to lowest within its own framework, with channel-evaluation items called out separately since their "score" is a recommendation, not a number comparable to RICE/ICE.

```
GROWTH BACKLOG — [date]

RICE-SCORED
# | Idea | Reach | Impact | Confidence | Effort | Score | Notes
1 | ...  | ...   | ...    | ...        | ...    | ...   | ...

ICE-SCORED
# | Idea | Impact | Confidence | Ease | Score | Notes

CHANNEL DECISIONS
Channel | LTV:CAC | Payback | Magic # | Quality flag | Recommendation
```

Flag anything that failed the Operating Rules gate (unsupported Confidence, missing inputs, wrong framework requested) inline rather than silently excluding it.

## Worked Example (Rivergate Data, fictional)

Marcus Iyer, Head of Growth Marketing, has three things on the table this week and wants them on one backlog, not scattered across a Slack thread — this is literally an open item on his stakeholder file: *"Wants the ICE/RICE growth-experiment backlog to be the single source of truth for what gets built next, not a Slack thread."*

**Item 1 — "Add a job-change signal alert to the outbound sequence trigger."** Marcus wants this piloted on one segment first (his standing ask), not shipped company-wide, so Reach is scoped to that segment. Est. Reach: 180 target accounts/quarter in the pilot segment. Impact: 2 (high, job-change is a proven trigger type elsewhere but unproven at Rivergate). Confidence: 60% — no Rivergate-specific data yet, based on published benchmarks from comparable B2B tools. Effort: 3 person-weeks (GTM engineering + one growth marketer).

RICE = (180 × 2 × 0.6) / 3 = **72**

**Item 2 — "Swap the demo-request CTA copy on the pricing page."** Impact: 6/10 (pricing-page CTAs are high-leverage but this is one line of copy, not a redesign). Confidence: 7/10 (two prior copy tests on this page moved conversion, this is a similar swap). Ease: 9/10 (one afternoon, no engineering).

ICE = 6 × 7 × 9 = **378** (normalized against other ICE items only, not compared directly to RICE scores)

**Item 3 — "Should we scale the LinkedIn outbound channel?"** This is a channel-spend question — Step 1 routes it away from RICE/ICE entirely.
- CAC: $1,850 (from Q2 spend ÷ closed-won attributed to the channel).
- LTV: $7,200 (Rivergate's blended LTV, no channel-specific adjustment yet — flagged as an estimate).
- LTV:CAC = 3.9:1 → healthy.
- Payback: 9 months → healthy.
- Quality check: 90-day retention for LinkedIn-sourced customers is in line with company average; no red flag.
- Magic Number: 0.68 (workable, not yet clearly efficient).
- **Recommendation: Test** — economics are healthy but Magic Number is borderline and the LTV figure is a blended estimate, not channel-specific. Before calling this Scale, get a channel-specific LTV pull (would come from the data warehouse integration named above) and re-run this check next quarter.

Also flagged: the pricing-page-change monitor Marcus asked for (competitor cut price 15% last quarter, caught via a lost deal, not monitoring) isn't a growth experiment at all, it's a competitive-intel gap. Noted here so it doesn't get lost, but routed out of this backlog to the monitoring skill that owns it.

```
GROWTH BACKLOG — 2026-07-02

RICE-SCORED
# | Idea                                  | Reach | Impact | Confidence | Effort | Score | Notes
1 | Job-change signal alert (pilot seg.)  | 180   | 2      | 60%        | 3      | 72    | Confidence estimated, no Rivergate data yet

ICE-SCORED
# | Idea                        | Impact | Confidence | Ease | Score | Notes
1 | Pricing-page CTA copy swap  | 6      | 7          | 9    | 378   | Backed by 2 prior tests on same page

CHANNEL DECISIONS
Channel            | LTV:CAC | Payback | Magic # | Quality flag | Recommendation
LinkedIn outbound  | 3.9:1   | 9mo     | 0.68    | none         | Test — LTV figure is blended, re-run channel-specific next quarter
```

## Rules

- Always route through Step 1 before scoring. A channel question scored on RICE is a misuse of the framework, not a shortcut.
- Never let Confidence or Impact carry a session on vibes alone, ask what evidence backs any input that looks inflated.
- RICE and ICE scores are never directly comparable to each other, keep them in separate backlog sections.
- A channel evaluation always ends in exactly one of Scale/Test/Kill/Invest-to-Learn, with the driving numbers stated, "it's complicated" is not a valid output.
- If Reach, Effort, CAC, or LTV inputs are missing, ask for them or mark the line `(estimated)`. Don't silently fill a gap with a plausible-sounding number.
- The output is always the full ranked backlog, not the single item that was just discussed, unless explicitly asked to score one item in isolation.
