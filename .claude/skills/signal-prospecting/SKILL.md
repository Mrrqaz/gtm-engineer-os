---
name: signal-prospecting
description: Multi-signal account scoring: job changes, funding announcements, hiring/departure signals, website-visitor identification, and tech-stack switches, each with a defined action window, combined into a tiered score that gates outreach priority. ICP fit gates the pipeline before any signal is scored. Outputs a structured signal card per qualified account. Run on a schedule per pilot segment, or on request for "what accounts should I be working this week."
user_invocable: true
---

# Signal Prospecting

Timing beats targeting. A perfectly-fit account with no live trigger converts worse than a mediocre-fit account mid-signal. This skill exists to catch the window, not just the fit: it gates accounts through ICP first, then scores whatever signals are actually firing, then tells me exactly what to do and by when before the window closes.

**Adapted from:** ColdIQ's `Signal Sourcer` and `List Building` master skills (github.com/sachacoldiq/ColdIQ-s-GTM-Skills): the job-change/funding/hiring/website-visitor/tech-change sub-skills with time-windowed action logic and multi-signal scoring tiers, and the 3-layer ICP framework with 100-point scoring from their `define-icp` skill. This version adapts that logic to Rivergate Data's pilot scope and this repo's evidence-gated, integrations-stated operating principles.

## Operating Rules (read first)

- **ICP gate comes before signal scoring, not after.** An account that fails ICP fit doesn't get scored on signals, no matter how strong the signal. A hot signal on a bad-fit account is noise, not a lead.
- **Never fabricate a signal.** If a data source isn't wired (see Intended Integrations), don't invent a job change, a funding round, or a tech-stack switch to fill out a card. Say the signal type wasn't checked this run.
- **Every signal has an expiry.** A signal outside its action window doesn't get a score contribution. Log it as expired, don't quietly keep counting it.
- **This pilot is scoped to one segment, per Marcus's standing request.** Don't expand the account list beyond the segment named in the run unless explicitly asked to widen it.
- **Scoring is additive and explainable.** Every point on a signal card traces to a named signal and a named rule. No black-box composite scores. Marcus has been burned by vendor attribution he couldn't audit, and this skill is built not to repeat that.
- **Outreach angle is a hypothesis, not a guarantee.** Frame the suggested angle as what the signal implies, not as confirmed intent.

## Intended Integrations (not live in this demo)

- **Clay** would run as the signal aggregation layer, pulling job-change, funding, and hiring-post data from its enrichment sources on a recurring table refresh, then pushing qualified rows into the CRM.
- **LinkedIn Sales Navigator API / Clay's LinkedIn integration** would power job-change and departing-employee detection (title-change and "no longer at" events on named target accounts).
- **A funding-data provider (e.g. Crunchbase API or Clay's funding source)** would power the funding-announcement signal.
- **A website-visitor identification tool (e.g. Clearbit Reveal, Koala, or RB2B)** would power the anonymous-visitor signal, matched against the ICP-gated account list, not run as an open firehose.
- **BuiltWith or Clay's tech-stack source** would power vendor-switch detection.
- None of these are wired into this repo. This is a demonstration of the scoring and gating logic, not a live pipeline. Every step below runs on data pasted into the session or already sitting in `context/`, and says so plainly when a signal type has nothing to check.

## Step 1: Confirm ICP Gate (3-Layer Framework)

Before any account enters signal scoring, run it through three layers. Score out of 100; **60+ required to proceed to signal scoring.** Anything below 60 gets logged as "gated out" with the layer that failed, and stops there.

**Layer 1: Firmographic fit (0-40 pts).** Company size, industry, tech stack compatibility, geography. Pull thresholds from `context/icp.md` if the repo has one seeded for this pilot; otherwise use the segment definition given at run time (e.g. "Series A-B B2B SaaS, 30-200 employees, US/UK, existing CRM in place").

**Layer 2: Behavioral fit (0-30 pts).** Evidence the account behaves like a buyer: content engagement, competitor usage, category-relevant hiring, prior inbound touch. This is about demonstrated interest in the problem space, separate from whether a specific signal fired this week.

**Layer 3: Timing fit (0-30 pts).** Budget cycle proximity, contract renewal windows, recent leadership change, stated growth plans. This layer is where signals will mostly land once scored in Step 2, but the *baseline* timing score here is about structural timing (fiscal year, renewal cycle), not the live event-based signals.

**Hard disqualifiers bypass the score.** If an account fails a segment-defining gate outright (below the pilot segment's employee floor, wrong geography, an explicitly excluded industry), it's gated out immediately regardless of what it might have scored on the other two layers. A 78/100 that sits below the employee floor is still out. Note the failing criterion and stop there.

Log the score and layer breakdown per account. Only accounts at 60+ (and past every hard disqualifier) move to Step 2.

## Step 2: Score Live Signals (Time-Windowed)

For each ICP-gated account, check for each signal type below. If the data source for a signal type wasn't provided this run, mark it "not checked", and don't assume absence means the signal didn't happen.

| Signal | Action window | Points | Note |
|---|---|---|---|
| Job change (new champion, VP+ into a relevant role) | 14-45 days post-change | +25 | ~3x response rate inside the window vs. cold; a new leader has budget authority and no incumbent-vendor loyalty for ~90 days, but is unreachable/defensive in the first 2 weeks |
| Funding announcement | 2-4 weeks post-announcement | +20 | Budget unlocks fast after the raise; waiting past week 4 means competing with every other vendor who also saw the TechCrunch post |
| Hiring signal, growth-indicating postings (e.g. 3+ open GTM/RevOps roles) | Active while postings are live | +15 | Growth hiring implies scaling pain in the exact function this pitch targets |
| Hiring signal, departing employee (former champion or economic buyer leaves) | 0-30 days post-departure | +15 (risk) or +10 (opportunity) | Risk framing if it's an existing account's champion leaving (churn signal); opportunity framing if it's a target account's incumbent-vendor advocate leaving (switch signal) |
| Website-visitor identification | 0-7 days post-visit | +15 | Decays fast. A week-old anonymous visit is cold by the time it's actioned |
| Tech-stack change (dropped or added a relevant vendor) | 0-60 days post-detected-change | +10 | Vendor churn is the longest window because stack changes are slower-moving and less time-sensitive than a personnel or funding event |

**Expired signals score 0** and get logged separately as "expired: was worth N pts, window closed [date]." This keeps the audit trail honest instead of quietly dropping them.

## Step 3: Apply Action Thresholds

Sum the signal points (ICP gate score is separate and already required 60+ to reach this step).

- **0 signal points** → not in this run's output. No live trigger, don't manufacture one.
- **1-14 points (single weak signal)** → **Watch.** Log the account, note the signal, no outreach yet. Re-check next cycle.
- **15-29 points (one strong signal, or two weak ones)** → **Prioritize.** Goes into this cycle's outreach queue at standard priority.
- **30+ points, or the named job-change + funding combo** → **Immediate outreach.** These get flagged at the top of the queue and the outreach angle should reference the specific signal, not a generic opener.

The named-combo callout exists because a job-change-plus-funding stack (new decision-maker with new budget) converts disproportionately better than either signal on its own. On the current point values it already clears the 30-point Immediate bar on raw sum (25 + 20 = 45), so the callout is not doing threshold-jumping work today. What it does guarantee is that this specific stack always lands at the very top of the Immediate queue, always gets signal-specific outreach rather than a generic opener, and stays Immediate even if the individual point weights get retuned later. ColdIQ's source material treats this combo as its own tier, and that priority treatment holds here.

## Step 4: Build the Signal Card

One card per qualified account (ICP 60+ and at least one non-expired signal). Format:

```
SIGNAL CARD - [Company Name]

ICP fit: [score]/100 (Firmographic [x]/40, Behavioral [x]/30, Timing [x]/30)
Signal score: [total] - [Watch | Prioritize | Immediate Outreach]

Signals fired:
- [Signal type]: [date detected] - [points] - window closes [date]
- [Signal type]: [date detected] - [points] - window closes [date]
(any expired signals logged separately below, not counted)

Primary contact: [name, title] [new-in-role flag if job-change signal]
Recommended action: [Watch / Prioritize this cycle / Immediate outreach - by when]
Suggested outreach angle: [one line tying the specific signal to a relevant
  problem, framed as hypothesis: "given X, they're likely feeling Y"]
```

## Step 5: Report the Run

Summarize the cycle: accounts checked, gated out at ICP (with layer breakdown counts), scored into Watch/Prioritize/Immediate, and any signal types not checked this run for lack of a data source. This is the diff the next cycle should build on, not a re-dump of every historical card.

## Worked Example (Rivergate Data pilot segment, fictional)

Marcus asked for signal-based prospecting piloted on one segment before company-wide rollout (`context/people/marcus-iyer-head-of-growth.md`, open items, 2026-06-19; adopted `context/decisions-log.md`, 2026-06-25). Segment for this pilot: **mid-market B2B SaaS accounts, 50-150 employees, that churned a competitor tool in the last 6 months**, Rivergate's strongest historical win pattern.

Pasted in for this run: three accounts with LinkedIn and Crunchbase data manually exported (no live Clay/LinkedIn API, see Intended Integrations).

```
Cycle summary - Rivergate Data pilot segment, week of 2026-06-29

Accounts checked: 3
Gated out at ICP: 1 (Northfield Ops, hard disqualifier: under 50 employees,
  below the pilot segment's floor, not scored further)
Signal types not checked this run: website-visitor identification, tech-stack
  change (no data source provided this run)

---
SIGNAL CARD - Vantree Analytics

ICP fit: 78/100 (Firmographic 34/40, Behavioral 24/30, Timing 20/30)
Signal score: 45 - Immediate Outreach (job change + funding combo)

Signals fired:
- Job change: 2026-06-14 - new VP Growth (ex-competitor-tool user) - +25 -
  window closes 2026-07-29
- Funding: 2026-06-20 - $12M Series A announced - +20 - window closes
  2026-07-18

Primary contact: Dana Okoro, VP Growth (new-in-role, 15 days)
Recommended action: Immediate outreach - send by 2026-07-05, before the
  job-change window's first-2-weeks defensiveness period fully clears but
  while still inside the 14-45 day response-rate window.
Suggested outreach angle: New VP Growth post-raise typically owns a fresh
  tooling decision in the first quarter. Hypothesis: Dana is likely
  evaluating whether the current stack scales with the new budget, worth
  testing an angle around headcount-efficient pipeline coverage, not a
  generic "congrats on the raise" opener.

---
SIGNAL CARD - Halloway Systems

ICP fit: 71/100 (Firmographic 30/40, Behavioral 21/30, Timing 20/30)
Signal score: 15 - Prioritize

Signals fired:
- Hiring signal (growth-indicating): 4 open RevOps/GTM postings live as of
  2026-06-27 - +15 - active while postings remain live

Primary contact: not yet identified. RevOps postings suggest a
  functional gap, not a named champion yet
Recommended action: Prioritize this cycle. Queue for standard-priority
  outreach; identify the hiring manager before sending.
Suggested outreach angle: Hypothesis: scaling GTM headcount without
  matching systems investment is a common gap at this stage, worth
  testing a systems-not-headcount framing once a contact is identified.

---
Expired signals this cycle: none.
```

## Rules

- ICP gate first, always. A hot signal never overrides a failed ICP gate.
- Never score a signal outside its action window. Log it as expired and move on.
- Never fabricate a signal type that wasn't actually checked this run. Say so plainly per signal type, not just once at the top.
- The named job-change + funding combo is always treated as top-priority Immediate Outreach, per Step 3. Don't extend that named treatment to other combos without the same evidence base.
- Keep the pilot scoped to the one segment Marcus named until he asks to widen it.
- Every outreach angle is a hypothesis framed as such, never asserted as confirmed intent, per Marcus's stated distrust of unverifiable attribution claims.
