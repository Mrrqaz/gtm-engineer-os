---
name: proof-boundary
description: The truth-status control that sits underneath every pipeline, revenue, and attribution number before it reaches an exec, the board, or a forecast call. Stamps each figure measured / projected / assumed / drafted, and enforces the two rules a GTM number most often breaks, never present a forecast as booked, and never claim a touch caused a conversion it cannot be traced to. Generalizes exec-revenue-briefing's "not reported this cycle" discipline into a repo-wide labeling pass. Run before any board deck, exec briefing, or forecast readout, or when asked "is this booked or forecast," "can we attribute that," "mark the truth-status," or "did that channel actually drive it."
user_invocable: true
---

# proof-boundary

GTM numbers fail in two specific ways, and both are the reason a revenue leader stops trusting the person who reports to them. A forecast gets repeated as if it were booked. A channel gets credited for a conversion it cannot actually be traced to. Both read confident, both compound, and both are hard to walk back once they land on a board deck. This skill exists to stamp the truth-status on every number before it gets there, so a projection is never read as a result and an attribution claim is never stronger than its evidence.

It labels; it does not build the briefing (that is `exec-revenue-briefing`), tier the deals (that is `pipeline-forecast-review`), or score the leads (that is `lead-routing`). It is the honesty pass those skills' outputs run through. It generalizes a rule `exec-revenue-briefing` already applies ("not reported this cycle") into a four-way label set every reported figure carries.

## The four labels

- **measured**: a confirmed, checkable figure. Closed-won ARR, actual sends, a CRM-confirmed stage change, a real event count. Present tense, backed by a source of record.
- **projected**: a modelled forward number. A forecast, pipeline coverage projected to quarter-end, a modelled attribution or a payback estimate. Never a result, and the label rides into every headline.
- **assumed**: an input taken on trust but not backed by evidence. A stage probability with no exit criterion met, a rep's verbal commit, an attribution inferred from timing rather than a traced touch. Usable only when labelled.
- **drafted / for-approval**: a number written into a briefing the exec has not yet confirmed or ratified.

## Operating Rules (read first)

- **Every figure carries a status.** A number with no label does not enter a briefing or a forecast readout. If I cannot say which of the four it is, it is not ready to report.
- **Never present a forecast as booked.** Coverage, forecast, and modelled numbers keep their `projected` label everywhere, including the one-line headline an exec repeats to the board. This is the rule that stops a pipeline number becoming a bookings claim in two hops.
- **Never claim an unearned attribution.** A channel or touch gets credit only for what can be traced to it. "The webinar drove three deals" is `assumed` at best unless there is a traceable touch on each of the three, in which case the honest version is "correlated, not yet causally traced." A caused-claim with no trace is blocked and restated.
- **A missing number is labelled, not filled.** `assumed` if it came from someone's word, `[NEED:]` if it came from nowhere. Never interpolated into something that reads clean. This is `exec-revenue-briefing`'s "not reported this cycle," applied to every figure.
- **I label, I do not shape, tier, or score.** Building the briefing is `exec-revenue-briefing`. Tiering deals against exit criteria is `pipeline-forecast-review`. Scoring leads is `lead-routing`. I stamp the truth-status of the numbers they produce. A well-built briefing can still carry a forecast mislabelled as booked, and catching that is my only job.

## Steps

1. Take each figure and each attribution claim in the output about to reach the exec (a briefing, a forecast readout, a QBR line).
2. Assign its truth-status from the four labels, grounded in the source behind it (a closed-won record, a forecast model, a verbal commit, a traced touch, or nothing).
3. Scan for the two GTM failures: a forecast written as booked, and an attribution claimed beyond what a traced touch supports. Restate the honest status for each.
4. Confirm every `projected` figure keeps its label in every place it appears, including summaries and headlines.
5. Hand the labelled set to `exec-revenue-briefing` (which shapes it and enforces no-orphaned-bad-news / no-surprises) and flag anything `assumed` or `[NEED:]` for confirmation before it goes to the board.

## Intended Integrations (not live in this demo)

- The `measured` figures would be checked against the **CRM** (HubSpot/Salesforce) for closed-won and stage changes, and a **data warehouse / BI view** for aggregated bookings, coverage, and attribution, so "measured" means checked against the source of record, not against a number pasted into a deck.
- **`pipeline-forecast-review` and `exec-revenue-briefing`** hand this skill their tiered deals and briefing figures; this skill stamps their truth-status before they land. None of it is wired here; inputs come from whatever the user provides in-session.

## Worked Example (fictional)

Rivergate Data, quarter-end board deck. `exec-revenue-briefing` has built it; I run the truth-status pass before Alex Novak (CEO) takes it to the board.

```
Rivergate Data - Board Deck Q2 - truth-status pass

- Closed-won ARR this quarter: $1.42M   [measured]  - CRM-confirmed, closed deals
- Forecast to quarter-end: $1.61M        [projected] - Commit + Best-Case, NOT booked
- Two late-stage deals ($180K) at 80%    [assumed]   - carried at 80% with no exit
                                                       criterion met; flagged upstream by
                                                       pipeline-forecast-review
- "June webinar drove 3 closed deals"                - BLOCKED. No traced touch on 2 of the
                                                       3. Restated below.
- SMB win rate: [NEED]                                - not reported this cycle, do not estimate
```

Two things I caught. The headline read "$1.61M closed this quarter," which is the forecast wearing a bookings label. I split it: "$1.42M booked, $1.61M forecast," so Alex never repeats a projection to the board as a result. And the marketing line crediting the June webinar with three deals had a traceable touch on only one of them, so I blocked the caused-claim and restated it as "1 of 3 has a traced webinar touch; the other 2 are correlated by timing, not attributed." Dana Whitfield (VP Sales) keeps the two 80% deals in the deck, but as `assumed`, tied to the exit-criteria gap `pipeline-forecast-review` already flagged, not as near-certain revenue.

I labelled every figure and held the two lines. I did not build the deck (`exec-revenue-briefing`), tier the deals (`pipeline-forecast-review`), or score anything (`lead-routing`).

## Adapted from

- The audit-trail / provenance idea, every reported figure keeps a reference to where it came from, is a pattern the sibling system repos in this set reuse per row. **VERIFIED in those builds**, not re-fetched here, so no star figure is attached. I turned it into a four-way truth-status label.
- The never-present-a-forecast-as-booked and never-claim-an-unearned-attribution rules are **my own design**, carried from the same control in those system repos and sharpened for GTM's specific failure mode (attribution inflation). This skill also generalizes a discipline already in this repo, `exec-revenue-briefing`'s "not reported this cycle," into a labeling pass every figure runs through.

## Rules

- Every figure and attribution claim carries a truth-status (measured / projected / assumed / drafted). An unlabelled number does not reach the exec.
- A forecast is never shown as booked. The `projected` label holds everywhere, including headlines.
- A channel gets credit only for a traced touch. A caused-claim with no trace is blocked and restated as correlated.
- A missing number is `assumed` (with a source to confirm) or `[NEED:]`, never estimated.
- I label. I do not build the briefing (`exec-revenue-briefing`), tier deals (`pipeline-forecast-review`), or score leads (`lead-routing`).
