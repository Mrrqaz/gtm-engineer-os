---
name: reply-triage
description: Classify every inbound reply from an outbound sequence into an 11-label taxonomy, roll up a positive_reply_rate against hard thresholds, and enforce an auto-pause safety rule when hostile or unsubscribe volume crosses a deliverability danger line. Route each label to the correct next action (AE handoff, suggested reply, record cleanup, or sequence stop). Run this whenever a batch of replies comes back from Smartlead, Instantly, an Apollo sequence, or Unipile LinkedIn — it's the skill that sits downstream of any send.
user_invocable: true
---

# Reply Triage

Sending is the easy part. What happens to a reply in the ten minutes after it lands is what decides whether a sequence keeps sending, gets throttled, or torches the domain's reputation. This skill takes a batch of raw replies, classifies each one against a fixed taxonomy, rolls the batch into one interest-rate metric, and enforces a hard safety brake before anything else gets sent.

**Adapted from:** the 11-label taxonomy, phrase-pattern rules, positive-reply-rate thresholds, and the hostile/unsubscribe auto-pause guardrail are adapted from `positive-reply-scoring/SKILL.md` in growthenginenowoslawski's `coldoutboundskills` repo (github.com/growthenginenowoslawski/coldoutboundskills, Smartlead-integrated). This skill is a re-implementation of that pattern for this operator OS, not original taxonomy design — credit stays with the source.

## Operating Rules (read first)

- **Every reply gets exactly one label.** If a reply could plausibly fit two labels, apply the priority order in Step 2 rather than guessing. Ambiguity is a reason to route conservatively (toward human review), not a reason to skip classification.
- **Auto-pause is a defensive stop, not a send.** Pausing a campaign to protect domain reputation is different in kind from sending a message on someone's behalf. It's reasonable to auto-execute because the downside of over-pausing (a delayed campaign) is far smaller than the downside of under-pausing (a burned domain). It still notifies a human immediately — auto-execute does not mean auto-silent.
- **Never draft or send on behalf of the AE for positive replies.** positive_interested and positive_soft route to the AE for a manual reply. This skill's job stops at classification and routing, not closing.
- **neutral_question gets a suggested draft, not a sent reply.** Draft it, flag it for approval, stop there.
- **negative_* always stops the sequence for that lead.** Never let a later touch in the same sequence fire after a negative classification, regardless of how the classification arrived (batch job or one-off).
- **Log the reason on every negative_notfit.** These feed ICP refinement. A negative_notfit with no logged reason is a wasted signal.
- **Don't compute positive_reply_rate on a sample too small to mean anything.** Below roughly 50 sends, report raw counts and hold off on the rate framing — a 1-in-20 batch is noise, not signal.

## Intended Integrations (not live in this demo)

- **Smartlead / Instantly reply webhooks** would feed Step 1 automatically — every inbound reply pushed to this skill the moment it lands, instead of waiting for a manual batch pull.
- **Apollo sequences** would be the source for sequences run through Apollo rather than Smartlead/Instantly, same webhook pattern.
- **Unipile** would feed LinkedIn DM/InMail replies into the same taxonomy, since a "reply" isn't only email — a LinkedIn response needs the same 11 labels and the same auto-pause guardrail.
- **Campaign pause API calls** (Smartlead `PATCH /campaigns/{id}/status`, Instantly's campaign pause endpoint) would execute Step 4's auto-pause for real. None of this is wired in this demo — replies are pasted or described in conversation, and "pause" means printing the flag and the notification, not calling an API.

## The 11-Label Taxonomy

Apply labels in this priority order — check from top to bottom, stop at the first match:

1. **unsubscribe** — Explicit opt-out. Phrases: "unsubscribe," "remove me," "take me off this list," "stop emailing me," a one-click unsubscribe link click. Always highest priority even if the same message contains other content.
2. **negative_hostile** — Anger, profanity, threats to report as spam, accusations of scraping/spam. Phrases: "how did you get my email," "this is spam," "reported," "f*** off," ALL CAPS hostility. Distinct from a plain no — the tone itself is the signal.
3. **bounce** — Automated delivery failure (hard or soft bounce), mailbox does not exist, mailbox full. Detected from the sending platform's bounce classification, not from message content.
4. **ooo** — Out-of-office autoresponder. Phrases: "out of office," "on leave until," "currently away," auto-reply header patterns. Time-boxed — always includes or implies a return date.
5. **negative_notfit** — Explicit no tied to a stated reason: wrong role, wrong company stage, no budget, no need, already has a solution. Phrases: "not a fit," "we already use X," "no budget for this," "wrong person, we don't do Y." The reason is the important part — log it.
6. **negative_notnow** — Explicit no with no reason, or a timing-only objection. Phrases: "not interested," "no thanks," "not right now," "maybe later," "check back in Q3." No stated reason to log beyond "timing."
7. **positive_referral** — Not the right person, but names or offers to loop in someone who is. Phrases: "not me, but try [name]," "loop in our [title]," "cc'ing our head of X." Route this as a warm handoff, not a dead end — closer to positive than negative.
8. **positive_interested** — Explicit interest, asks for a call/demo/pricing, or a clear buying signal. Phrases: "let's chat," "send me a time," "interested, tell me more," "what does pricing look like," a Calendly link click plus a reply.
9. **positive_soft** — Curious but non-committal. Engagement without a clear next-step ask. Phrases: "tell me more," "interesting, what exactly does this do," a question about the product with no scheduling language. Softer than positive_interested — needs a human touch to convert, not a booking link.
10. **neutral_question** — A factual or clarifying question with no interest signal either way. Phrases: "how did you get my info," "who are you," "is this from a person or a bot," "what company are you with." Answerable without committing either party to anything.
11. **other** — Doesn't cleanly fit any label above: forwarded internally with no reply text, garbled/spam-filtered content, a reply in a language the pipeline can't classify confidently. Default label of last resort — flag for human review rather than force-fitting one of the ten labels above.

## Step 1: Pull the Reply Batch

Take whatever replies the user has pasted or described for the period in question (a day, a week, a specific campaign). Each reply needs: sender, company, the reply text itself, and — if available — total sends for the campaign in the same window (needed for Step 3's rate).

If total sends isn't available, still classify every reply; just note that the rollup rate can't be computed and report raw label counts instead.

## Step 2: Classify Each Reply

Walk the priority list above top to bottom for each reply. Record the label and the specific phrase or signal that triggered it — don't just assert a label, show the evidence in one short quote or paraphrase. This is what makes the classification auditable instead of a black box.

If a reply is genuinely borderline between two labels, route to the more conservative one:
- Between positive_soft and neutral_question → neutral_question (don't oversell interest).
- Between negative_notnow and negative_hostile → negative_hostile only if there's an actual tone signal, not just a blunt no.
- Between other and anything else → prefer the specific label; other is for genuine no-fits, not a shortcut past classification effort.

## Step 3: Roll Up positive_reply_rate

```
positive_reply_rate = (positive_interested + positive_soft) / total_sent
```

Interpretive thresholds (hard, not vibes):

| Rate | Read |
|---|---|
| < 1% | Below floor — sequence, list quality, or ICP targeting needs a look before pushing more volume |
| ≥ 1% | Good — sequence is working, keep sending at current volume |
| ≥ 2% | Great — strong signal to increase volume or replicate the angle across adjacent segments |

Report the rate alongside the raw counts (X positive / Y total sent), never the rate alone — a rate with no denominator context is not verifiable.

## Step 4: Auto-Pause Safety Check

This runs on every batch, independent of Step 3, and it is the part of this skill that actually protects the sending domain.

```
hostile_rate = negative_hostile / total_sent
unsubscribe_rate = unsubscribe / total_sent
```

**Auto-pause triggers if either:**
- `hostile_rate > 0.3%`, OR
- `unsubscribe_rate > 2%`

If triggered:
1. Flag the campaign as **PAUSED — AUTO** in the output.
2. State which threshold was crossed and the exact numbers (e.g. "hostile_rate 0.41% (4/975) exceeds the 0.3% guardrail").
3. Notify a human immediately as part of the same output — this is not a silent background action. The notification is the point; auto-executing the pause without surfacing it defeats the safety purpose.
4. Recommend what to check before resuming: list quality, subject line/opener tone, whether a specific segment or list source is driving the hostile replies disproportionately.

This threshold check runs **before** rate framing from Step 3 gets any weight in the recommendation — a campaign can have a great positive_reply_rate and still need to pause if it's also burning the domain's reputation on the way there. Good conversion doesn't buy back a reputation problem.

## Step 5: Route Each Label

| Label(s) | Route |
|---|---|
| positive_interested, positive_soft | AE for manual reply. Hand off with the lead's full context, not just the label — the AE needs the original message, not a tag. |
| positive_referral | AE for manual reply, flagged as a warm handoff to a named contact — don't let this get buried as a soft positive. |
| neutral_question | Draft a suggested reply, flag for approval. Never send a neutral_question reply without a human sign-off — the wrong tone here can turn a neutral into a negative_hostile fast. |
| ooo | Reschedule the touch for after the stated return date. Don't advance the sequence step, just delay it. |
| bounce | Clean the record — mark the contact undeliverable, remove from active sequencing, flag for re-verification before any future campaign reuses the list. |
| negative_notnow, negative_notfit, negative_hostile, unsubscribe | Stop the sequence for this lead immediately. No further touches. |
| negative_notfit specifically | Also log the stated reason in a running ICP-refinement note — this is the input that improves targeting for the next campaign, not just a closed record. |
| other | Flag for human review. Don't force it into a route; a human decides what "other" actually needs. |

## Worked Example (Rivergate Data, fictional)

Rivergate Data (Series A B2B SaaS) ran an outbound sequence to 620 prospects this week. 14 replies came back.

```
REPLY TRIAGE — Rivergate Data outbound, week of 2026-06-29
Total sent: 620 | Replies: 14

1. Priya Nandan (VP Ops, Delford Logistics) — "This is actually well timed,
   can you send over some times next week?"
   → positive_interested ("send over some times" = scheduling ask)
   → Route: AE manual reply, full context handed off

2. James Okoro (RevOps Lead, Marbury) — "Interesting, what exactly does
   this plug into on our side?"
   → positive_soft (curiosity, no scheduling ask)
   → Route: AE manual reply

3. Dana Reyes (COO, Fennimore) — "Not the right person for this, try our
   Head of Sales Ops, cc'd here."
   → positive_referral (names + loops in a specific contact)
   → Route: AE manual reply, flagged as warm handoff to named contact

4. Unnamed sender, Torvalt Inc — "How did you get this email address?"
   → neutral_question (factual question, no interest signal either way)
   → Route: suggested reply drafted, flagged for approval

5. Auto-reply, Kestrel Group — "I am out of office until July 14th."
   → ooo
   → Route: reschedule touch to July 15

6-7. Two hard bounces (mailbox does not exist), platform-flagged.
   → bounce (x2)
   → Route: records marked undeliverable, removed from active sequence

8. Marcus Webb (Founder, Aldergate Studio) — "Not interested, we built
   this in-house already."
   → negative_notfit (stated reason: already has a solution)
   → Route: sequence stopped, reason logged for ICP refinement

9. Two "no thanks, not right now" replies with no reason given.
   → negative_notnow (x2)
   → Route: sequence stopped, no reason to log beyond timing

10. Unnamed sender — "STOP EMAILING ME, this is spam, I'm reporting this."
   → negative_hostile ("this is spam" + reporting threat, not just a no)
   → Route: sequence stopped immediately

11. One unsubscribe-link click, no message text.
   → unsubscribe
   → Route: sequence stopped, suppressed from future sends

12. One forwarded-internally reply with no reply text of its own.
   → other
   → Route: flagged for human review

ROLLUP
positive_reply_rate = (1 positive_interested + 1 positive_soft) / 620
                     = 2 / 620 = 0.32%
Below the 1% floor. Positive volume this week is thin relative to send
volume — worth reviewing whether this list/segment matches the ICP that's
converted before, rather than just pushing more volume at the same list.

AUTO-PAUSE CHECK
hostile_rate = 1 / 620 = 0.16% — under the 0.3% guardrail, no pause
unsubscribe_rate = 1 / 620 = 0.16% — under the 2% guardrail, no pause
Campaign stays live. Both rates are well inside the safe range; the
positive_reply_rate issue above is a targeting problem, not a
deliverability problem.
```

If the hostile count in this batch had been 3 instead of 1 (`3/620 = 0.48%`), Step 4 would have triggered: **PAUSED — AUTO, hostile_rate 0.48% exceeds 0.3% guardrail**, with an immediate notification and a recommendation to check whether one specific list source is driving the hostile replies before resuming.

## Rules

- Classify against the priority-ordered taxonomy every time — don't skip straight to "positive or negative," the routing depends on the specific label.
- Show the triggering phrase or signal for every classification. An unauditable label is not a usable label.
- Auto-pause is allowed to execute without waiting for approval, because it's a stop, not a send — but it must always come with an immediate human notification in the same output, never silently.
- Never let the positive_reply_rate framing soften or delay the auto-pause check. Check the guardrail first, report the rate second.
- negative_notfit reasons get logged for ICP refinement every time, not just when convenient.
- Below ~50 sends, hold off on rate framing and report raw counts instead — small-sample rates create false confidence in either direction.
