---
name: lifecycle-signal-tracker
description: Post-sale lifecycle signal tracking across the whole book — churn risk AND expansion/upsell readiness in one pass, since they're two sides of the same lifecycle-health coin. Builds a chronological signal timeline across five domains (adoption, engagement, support, commercial, renewal), classifies churn root cause against a fixed taxonomy with confidence tiers, and computes a pace multiplier against the onboarding/adoption plan to flag accounts falling behind or running ahead of schedule. Three modes: --account (single deep-dive), --portfolio (full book scan), --patterns (systemic pattern search). Run after a QBR, before a renewal conversation, when a rep flags a worried account, or on a standing weekly/monthly cadence across the whole book.
user_invocable: true
---

# Lifecycle Signal Tracker

Churn analysis and time-to-value analysis are usually built as two separate tools, run by two different teams, at two different points in the customer lifecycle. That's a mistake — both are reading the same underlying signal (is this account's pace of value realization on track), just from opposite ends. An account that's falling behind plan is a churn risk. An account that's running ahead of plan is an expansion-ready signal. This skill runs both passes together so a single scan of the book surfaces at-risk and expansion-ready accounts side by side, instead of needing two tools and two owners to see the whole picture.

## Operating Rules (read first)

- **The CRM's logged "close reason" is not ground truth.** Reps mis-log close reasons constantly — under time pressure, to avoid a hard conversation with their manager, or because the real reason surfaced after the loss was already logged. Treat the CRM field as one input to corroborate, never as the answer.
- **Don't over-weight the most recent event.** A support ticket filed the week before a renewal conversation feels causal but usually isn't the root cause — it's often the last visible symptom of a pattern that's been building for months. Build the full timeline before naming a cause.
- **Book-wide pattern matches are leads, not verdicts.** If this account's signal pattern resembles three other accounts that churned last quarter, that's a reason to dig deeper on this account, not a reason to write the risk brief before digging.
- **Confidence tiers are driven by corroboration, not conviction.** A root-cause call is only as strong as the number of independent sources that agree. One source, however strong-sounding, does not earn High confidence.
- **Pace-multiplier output never goes to the customer.** It's an internal planning signal computed against Rivergate's own onboarding plan, not a claim about the customer's competence or effort. It gets tagged `[review — internal planning target]` everywhere it appears and stays in internal briefs only.
- **Portfolio pattern claims need 5+ accounts.** Below that, the finding is real but the sample is too small to generalize — say "directional, not confirmed" and name the actual count. Don't round 3 accounts up to "a pattern."
- **Segment-normalize before judging pace.** An Enterprise account moving at SMB pace looks alarming and isn't — the plans have different expected timelines. Always compare pace against the plan for that account's own segment.

## Intended Integrations (not live in this demo)

- **Product-usage telemetry (Amplitude)** would feed the adoption and engagement domains automatically — feature activation rates, login frequency, seat utilization — instead of relying on manually pasted usage summaries.
- **Support-ticket system (Zendesk/Intercom)** would feed the support domain — ticket volume, severity trend, CSAT on resolved tickets, escalation flags.
- **CRM (HubSpot/Salesforce)** would supply the commercial and renewal domain data (contract value, renewal date, logged close/expansion reasons) and the onboarding/adoption plan milestones used for the pace-multiplier baseline.
- None of this is wired in this repo. Every mode below runs on whatever timeline data, usage summaries, or transcripts the user pastes in, or states plainly that a domain has no data and is being scored on partial evidence. The classification taxonomy, confidence-tier logic, pace-multiplier formula, and guardrails are unchanged whether the input is pasted text or a live integration.

## Step 1: Determine Mode

Read the invocation flag:

- `--account <name>` — single-account deep dive. Produces a risk/opportunity brief.
- `--portfolio` — scans every account with available data. Produces a portfolio scan flagging both at-risk and expansion-ready accounts in one pass.
- `--patterns` — looks across the whole book for recurring signal combinations (e.g., "accounts with a champion change inside 60 days of renewal churn at 3x the base rate"). Only run this after `--portfolio` has been run at least once, it needs the classified account set to look for patterns across.

If no flag is given, ask which mode before doing anything else — the three modes have different evidence bars and shouldn't be conflated.

## Step 2: Build the Signal Timeline (per account)

For the account(s) in scope, pull every available signal into one chronological timeline, tagged by domain:

1. **Adoption** — feature activation, seat utilization, onboarding-milestone completion dates.
2. **Engagement** — login frequency, meeting attendance/no-shows, responsiveness to outreach.
3. **Support** — ticket volume and severity trend, escalations, CSAT.
4. **Commercial** — contract value changes, invoice/payment friction, procurement contact changes.
5. **Renewal** — renewal date proximity, logged renewal conversations, stated intent from the account team.

Order every entry by date, not by domain. The point of a single chronological view is that cross-domain sequencing is itself diagnostic — a champion departure (engagement) followed six weeks later by a support escalation (support) followed by a downgrade request (commercial) tells a different story than the same three events happening in isolation on the same week.

If a domain has no data available for this account, say so explicitly in the timeline output rather than silently omitting it — a blank support domain might mean "no issues" or might mean "we don't have ticket data for this account," and those are very different states.

## Step 3: Classify Root Cause (churn side)

Classify the dominant risk driver against this fixed taxonomy. An account can have more than one driver — name the primary and any secondary.

| Category | What it looks like |
|---|---|
| **Adoption Failure** | Low usage, incomplete onboarding, feature activation stalled — the product isn't being used enough to prove value |
| **Relationship Gap** | Champion departure, no executive sponsor, engagement dropped after a key contact left |
| **Commercial Misalignment** | Price sensitivity, budget cuts, procurement pushback, contract terms friction |
| **Product Gap** | Repeated feature requests going unmet, competitor has a capability Rivergate doesn't, workaround fatigue |
| **Competitive Displacement** | Explicit mentions of a competitor evaluation, RFP activity, migration signals |
| **Force Majeure** | Company-level event unrelated to the product — layoffs, acquisition, budget freeze, business closure |

**Confidence tiers** — set by how many independent sources corroborate the classification, not by how compelling any single source sounds:

- **High** — 3+ independent sources agree (e.g., a support ticket pattern, a declining usage trend, and a direct statement from the customer all point the same direction).
- **Moderate** — 2 independent sources agree.
- **Low** — 1 source, or sources that only weakly correlate. State the classification as a hypothesis, not a finding, and name what additional evidence would raise confidence.

Explicitly check the CRM's logged close/risk reason against the timeline you built. If they disagree, say so — that disagreement is itself useful information about where the account team's read differs from the evidence.

## Step 4: Compute the Pace Multiplier (expansion side)

For accounts with an onboarding/adoption plan on file:

```
Pace Multiplier = Actual Days Elapsed ÷ Planned Days Elapsed (at the same milestone)
```

- **> 1.15** — at-risk: the account is taking meaningfully longer than planned to reach this milestone. Falls into the churn-side analysis too — check whether Adoption Failure is the driver.
- **0.85–1.15** — on pace, no flag.
- **< 0.85** — ahead of plan: potential expansion-ready signal. Fast time-to-value is one of the strongest predictors of expansion readiness — flag for the account team to explore an upsell or expansion conversation.

**Always normalize by segment before applying these thresholds.** SMB and Enterprise plans have different expected paces built into their onboarding plan (Enterprise plans are typically longer and more milestone-heavy) — the multiplier is only meaningful when compared against the plan for that account's own segment, never against a blended average.

Tag every pace-multiplier output `[review — internal planning target]`. This number describes Rivergate's plan versus reality, not the customer's performance, and it is never customer-facing.

## Step 5: Portfolio Scan (`--portfolio` mode)

Run Steps 2–4 across every account with available data. Output two lists in one pass:

- **At-risk** — accounts with a Moderate+ confidence churn classification, or a pace multiplier > 1.15.
- **Expansion-ready** — accounts with a pace multiplier < 0.85 and no offsetting churn risk signal in the same timeline.

An account can theoretically appear on neither list (on pace, no risk signal) — don't force every account into one bucket or the other.

## Step 6: Pattern Search (`--patterns` mode)

Look across the classified portfolio set for recurring signal combinations — the same root-cause category clustering in a segment, a timeline sequence (e.g., champion change → support escalation → downgrade) repeating across multiple accounts, a pace-multiplier band correlating with a specific plan type.

**Minimum 5 accounts required to state a pattern.** Below 5, report the observation but downgrade the language explicitly: "directional, not confirmed — only 3 accounts show this sequence." Never round up to "a pattern" language on a small sample, and never let a 5-account pattern get reported with the same confidence as a 20-account one — state the n every time.

## Step 7: Output

**Single-account mode** produces a risk/opportunity brief:

```
LIFECYCLE BRIEF — [Account]  ([Segment])

TIMELINE
[chronological signal list, tagged by domain, with dates]

CHURN READ
Primary driver: [taxonomy category]  |  Confidence: [High/Moderate/Low]
Secondary driver (if any): [category]
CRM logged reason: [what's in the CRM] — [agrees / disagrees] with timeline read
Evidence: [the specific corroborating sources, named]

PACE READ
Pace multiplier: [X.XX]  [review — internal planning target]
[flag: at-risk / on-pace / expansion-ready, with the milestone it's measured against]

RECOMMENDATION
[one paragraph: what the account team should do next, and why —
never "monitor," always a concrete next action]
```

**Portfolio mode** produces a two-list scan:

```
PORTFOLIO SCAN — [date]

AT-RISK  ([count] accounts)
[Account] — [root cause, confidence] — [pace multiplier if relevant]
...

EXPANSION-READY  ([count] accounts)
[Account] — pace [X.XX] [review — internal planning target] — [why it's expansion-ready]
...

NO FLAG  ([count] accounts, one line, don't detail each)
```

## Worked Example (Rivergate Data, fictional)

Marcus Iyer asked on 2026-06-25 for this exact behavior: he wants the churn taxonomy to also flag expansion-ready accounts, not just at-risk ones, because his team was only ever getting the bad-news half of the picture from the old churn tool. Running `--portfolio` on Rivergate's mid-market book:

```
PORTFOLIO SCAN — 2026-07-02

AT-RISK  (2 accounts)
Brightline Ops — Adoption Failure, Moderate confidence (usage flat for 6
  weeks, onboarding milestone 3 still incomplete at day 74 of a planned
  45) — pace 1.64 [review — internal planning target]. CRM logged reason
  was "pricing concern" from the last QBR note; timeline disagrees —
  no commercial-domain signal until after the usage stall was already
  visible. Flag the CRM note as likely a symptom, not the cause.
Halston Freight — Relationship Gap, Low confidence (champion listed as
  "no longer with company" in one CRM field update, no corroborating
  engagement-domain drop yet, no support-domain signal either) — this is
  a lead to check, not a finding. One source only.

EXPANSION-READY  (1 account)
Wovenpath Labs — pace 0.71 [review — internal planning target], hit
  onboarding milestone 4 in 32 days against a 45-day SMB plan, high
  engagement domain activity (weekly logins from 4 seats vs. the 2-seat
  contract). Flag for the account team: fast pace plus usage above
  contracted seats is a clean expansion-conversation trigger.

NO FLAG  (14 accounts) — on pace, no churn signal above Low confidence.
```

Note the Halston Freight entry: it's a real lead (a champion-change signal did surface), but it stays Low confidence and gets described as "a lead to check" because it's one source with nothing corroborating it yet — exactly the guardrail against turning a single data point into a verdict.

## Rules

- Never treat the CRM's logged close/risk reason as the answer. It's one input to check the timeline against, and disagreement between the two is itself worth reporting.
- Never let the most recent event anchor the root-cause call. Build the full timeline first, then classify.
- Confidence tiers are set by source count, not by how convincing any one source sounds. One strong-sounding source is still Low confidence.
- Book-wide pattern matches are investigation leads. Don't write a final risk verdict off a resemblance to other churned accounts, dig into this account's own timeline first.
- Pace-multiplier output is always tagged `[review — internal planning target]` and never appears in anything customer-facing.
- Portfolio-level pattern claims require 5+ accounts. Below that, say "directional, not confirmed" and state the actual n.
- Always normalize pace against the plan for the account's own segment. Never compare across segments on a blended baseline.
- If a domain has no data for an account, say so explicitly rather than silently treating it as "no risk in that domain."
- The recommendation line is mandatory in single-account mode and must be a concrete next action, not "keep monitoring."
