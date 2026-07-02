---
name: campaign-experiment-design
description: Designs outbound campaign A/B tests (list-only, copy-only, or combined) with a pre-launch sample-size gate, pre-declared success/failure/inconclusive criteria, and confidence-weighted results reporting. Run before launching any outbound experiment, and again when results come in to grade the read against what was pre-declared. Adapted from growthenginenowoslawski/coldoutboundskills' experiment-design skill.
user_invocable: true
---

# Campaign Experiment Design

Most outbound "tests" aren't tests. Someone changes the list and the copy at the same time, eyeballs the reply rate after three days, and calls it a win. This skill exists to stop that: one variable at a time unless time-boxed otherwise, a sample-size check before launch (not after), and a written definition of win/lose/inconclusive before a single email goes out. The discipline is deciding what counts as a result before you've seen any results.

**Adapted from:** growthenginenowoslawski/coldoutboundskills, `experiment-design/SKILL.md` (github.com/growthenginenowoslawski/coldoutboundskills). The three experiment types, the sample-size gate, the pre-declared-criteria discipline, and the confidence-weighted results schema are that skill's design. This version adapts it to this repo's stakeholder (Marcus Iyer, Head of Growth Marketing) and worked example (Rivergate Data).

## Operating Rules (read first)

- **One variable at a time, by default.** List-only and copy-only are the two clean experiment types. Combined (both list and copy change) is allowed only when explicitly time-constrained, and every combined result gets flagged as harder to attribute — never reported with the same confidence as a single-variable test.
- **No launch without a sample-size check.** Before any test goes live, estimate baseline reply/open rate from historical data for that segment and confirm the available volume can plausibly reach a usable read in the test window. If it can't, don't launch it as a test — say so and propose a different question.
- **Success/failure/inconclusive get defined before launch, in writing.** Not after looking at the numbers. If someone asks "so did it work?" and the answer requires deciding *right then* what would have counted as working, the test wasn't designed, it was run.
- **Every conclusion carries a confidence tag.** HIGH, MEDIUM, or LOW, based on whether the actual sample reached matches what was pre-declared as the minimum. A result on an underpowered sample is still reportable, it just can't be reported as if it weren't underpowered.
- **This is deliberately not a full stats engine.** The sample-size table below is illustrative and directional, built for outbound volumes and go/no-go decisions, not for publishing a p-value. If a call is genuinely borderline, extend the test window before trusting a marginal read.

## Step 1: Pick the Experiment Type

Ask what's actually being tested, then name the type explicitly, don't leave it implicit.

- **List-only**: same copy, sent to two different segments or lists. Answers "does this message work better on a different audience?" Use when copy already has a working baseline and the open question is targeting.
- **Copy-only**: same list, two different messages. Answers "does a different angle/hook/CTA move this specific audience?" Use when the list is fixed (limited volume in an ICP segment, or a segment already committed to) and the open question is messaging.
- **Combined**: both list and copy differ. Only used when time-constrained (e.g., a launch deadline that can't wait for a clean sequential test). Flag combined tests as **attribution-weak** in every downstream report, if the segment wins, you won't know whether it was the list or the copy without a follow-up single-variable test.

Default to list-only or copy-only. If someone proposes changing both "to save time," name the tradeoff out loud before agreeing to it.

## Step 2: Estimate Baseline Rate

Pull the historical reply rate (or open rate, if reply rate isn't the metric in play) for the segment or list being tested, from prior sends in the same or a comparable segment. This is the number every sample-size calculation below depends on.

**Sanity floor check:** if the estimated baseline is at or below roughly 1%, treat that as a hard flag before proceeding. At sub-1% baselines, detecting anything short of a very large lift requires sample sizes most outbound lists can't produce in a reasonable window. Either the test needs a much bigger lift threshold, a much longer window, or it isn't a good candidate for a formal test right now, it's a candidate for a bigger structural change first (list quality, ICP fit, offer), re-tested once the baseline moves.

If no historical data exists for this exact segment, use the closest comparable segment and say explicitly that the baseline is an estimate, not a measured number. Don't invent a baseline.

## Step 3: Check Sample Size Against Expected Lift

Bigger claimed lifts need smaller samples to detect; smaller claimed lifts need much bigger ones. This table is illustrative, not a stats engine, use it for a go/no-go read, not a footnote-worthy citation.

| Expected lift (relative) | Approx. minimum sample per variant* | Read |
|---|---|---|
| 2x (100% relative lift) | ~100-150 | Easy to detect, most lists can hit this in a normal send window |
| 50% relative lift | ~300-500 | Moderate, needs a real list or a longer window |
| 20% relative lift | ~1,000-1,500 | Hard, only realistic on larger lists or multi-week windows |
| 10% relative lift or smaller | ~3,000+ | Usually not worth testing formally on outbound volumes, treat as directional at best |

*Rough figures assuming a baseline reply rate in the low single digits (2-5%). Lower baselines push every row up; higher baselines pull every row down. Recompute directionally against Step 2's actual baseline rather than treating this table as fixed.

Compare the table's minimum against what the segment or list can actually deliver in the intended test window. If the available volume can't reach the minimum for the lift being claimed, that's the sample-size gate failing, don't launch as designed. Either lower the claimed lift threshold to something the volume can detect, extend the window, or combine adjacent segments if that doesn't corrupt the question being asked.

## Step 4: Pre-Declare Success, Failure, and Inconclusive

Before launch, write down all three, not just what a win looks like:

- **Success**: the specific threshold that counts as a real, actionable win (e.g., "variant B reply rate is at least 50% relative higher than variant A, with both variants having reached the Step 3 minimum sample").
- **Failure**: the specific threshold that counts as a real, actionable loss (e.g., "variant B reply rate is equal to or below variant A after both reach minimum sample" — not "worse by some amount," equal-or-worse is still informative and should be captured, not thrown away as noise).
- **Inconclusive**: what happens if the test window closes before either variant reaches the Step 3 minimum sample, or the result sits inside a range too close to call either way. This is the outcome people forget to define, and it's the one that gets argued about after the fact if it isn't written down first.

This declaration gets written into the test record before the first send, not drafted after the numbers come in. If the question "did it work" can only be answered by relitigating what "work" means, the test wasn't designed.

## Step 5: Launch, Then Report With Confidence Tags

Once live, track actual sample reached per variant against the Step 3 minimum. When reporting results (interim or final), tag every conclusion:

- **HIGH confidence**: both variants reached or exceeded the pre-declared minimum sample, and the result clearly sits on one side of the pre-declared success/failure line.
- **MEDIUM confidence**: at least one variant is close to but hasn't fully reached the pre-declared minimum, or the result is directionally clear but inside a range that's still somewhat sensitive to a few more replies either way.
- **LOW confidence**: sample reached is well under the pre-declared minimum (interim read, early days of the test, or a segment that under-delivered volume), or it's a combined (list+copy) test where the win can't be attributed to either variable cleanly. LOW confidence results are still worth reporting, they just can't be acted on as if they were HIGH.

Results schema for the written report:

```
Test: [name]
Type: [list-only | copy-only | combined]
Segment(s)/List(s): [...]
Variants: [A description] vs [B description]
Baseline rate used: [X%, source: measured on <segment/date range> | estimated from <comparable segment>]
Pre-declared minimum sample per variant: [N]
Pre-declared success: [threshold]
Pre-declared failure: [threshold]
Pre-declared inconclusive condition: [threshold/window]

Actual sample reached: A=[n], B=[n]
Actual result: A=[rate], B=[rate]
Verdict: [Success | Failure | Inconclusive]
Confidence: [HIGH | MEDIUM | LOW] — [one line why]
Attribution note: [only for combined tests — what follow-up single-variable test would confirm which variable drove it]
```

## Intended Integrations (not live in this demo)

- **Smartlead / Instantly** would be the send platforms for both variants, and the source of actual reply/open counts per variant during the test window. Right now this skill runs on pasted or manually pulled send data, not a live campaign API.
- **HubSpot workflows** would tag contacts by variant at enrollment and hold the segment split steady for the duration of the test, so a mid-test list change (a new lead added to the segment) doesn't quietly contaminate the sample. Not wired here.
- **A shared experiment log** (a HubSpot custom object, an Airtable base, or similar) would hold every test's pre-declared record from Step 4 before launch, so results can be graded against what was actually declared rather than a reconstructed memory of it. This demo treats the Step 5 schema as the record; in production it would be written to that log at launch time, before any data comes in.

## Worked Example (Rivergate Data, fictional)

Marcus Iyer wants to test whether leading with a funding-signal hook out-performs the current generic cold-open on Rivergate's mid-market SaaS segment.

**Step 1 — type:** Copy-only. The list (mid-market SaaS, ICP-fit score >= 7) stays fixed; only the opening line changes. Variant A is the current generic open, Variant B leads with "saw you just raised your Series B."

**Step 2 — baseline:** Pulling the last 90 days of sends to this segment, reply rate has been running ~3.5%. That's comfortably above the 1% sanity floor, so the test is a real candidate.

**Step 3 — sample size:** Marcus wants to detect at least a 50% relative lift (3.5% -> ~5.25%+). Per the table, that needs roughly 300-500 sends per variant. Rivergate's mid-market SaaS segment only has ~450 leads total that qualify this month, well short of 600-1,000 combined across two variants in a normal window.

This is the exact gate Marcus pushed back on in the 2026-06-18 review of this skill (see `context/decisions-log.md` and his file in `context/people/`): the default minimum was calibrated for a larger list than Rivergate actually has. Rather than skip the gate, the threshold gets adjusted per-segment: for this segment, the test is redesigned to detect a 2x lift instead of 50% (needing only ~100-150 per variant, achievable within the available list), and the test window is extended from two weeks to four to let both variants actually clear that smaller minimum. The gate itself doesn't move, the target lift and the window do.

**Step 4 — pre-declared:**
- Success: Variant B reply rate is at least 2x Variant A's, with both variants at or above 120 sends.
- Failure: Variant B reply rate is equal to or below Variant A's after both reach 120 sends.
- Inconclusive: the four-week window closes with either variant under 120 sends, or Variant B is directionally higher but under the 2x threshold.

**Step 5 — result (illustrative):** After four weeks, A reached 135 sends at 3.0% reply, B reached 128 sends at 7.8% reply — a 2.6x lift, clearing the pre-declared 2x success threshold with both variants past the adjusted 120-send minimum.

```
Test: Funding-signal open vs generic open — mid-market SaaS
Type: copy-only
Segment(s)/List(s): mid-market SaaS, ICP score >= 7
Variants: A) generic cold-open  B) funding-signal-led open
Baseline rate used: 3.5%, measured on trailing 90-day sends to this segment
Pre-declared minimum sample per variant: 120 (adjusted from default 300-500 per Marcus's 2026-06-18 segment-size pushback)
Pre-declared success: B reply rate >= 2x A's, both variants >= 120 sends
Pre-declared failure: B reply rate <= A's, both variants >= 120 sends
Pre-declared inconclusive condition: either variant under 120 sends at window close (4 weeks), or B directionally higher but under 2x

Actual sample reached: A=135, B=128
Actual result: A=3.0%, B=7.8%
Verdict: Success
Confidence: HIGH — both variants cleared the pre-declared 120-send minimum and the result sits clearly past the 2x threshold, not a borderline read
Attribution note: n/a — copy-only test, list held constant
```

Next step flagged to Marcus: this validates the funding-signal hook on mid-market SaaS specifically. Rolling it out to other segments is a new test, not an assumption, the lift doesn't automatically transfer to a segment with a different baseline or buyer profile.

## Rules

- Never let a test launch without a written pre-declared success/failure/inconclusive definition. If it isn't written down before the first send, it doesn't count as a designed test, it's a retrospective story.
- Never report a result without a confidence tag. An underpowered read is still worth sharing, but it has to say so.
- Never treat a combined (list+copy) test result as attributable to either variable alone. Flag it and name the follow-up single-variable test that would confirm it.
- Never skip the sanity-floor check on baseline rate. A test that mathematically can't reach a usable sample in the available window isn't a test, it's wasted list.
- The sample-size table is a go/no-go tool, not a citable statistic. When a call is genuinely borderline, extend the window before trusting a marginal read, don't round in favor of the answer someone wants.
- If a stakeholder wants to change a pre-declared threshold after seeing early results, that's a new test with a new declaration, not an edit to the old one. Say so plainly.
