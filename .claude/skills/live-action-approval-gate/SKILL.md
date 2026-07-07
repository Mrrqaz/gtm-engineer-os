---
name: live-action-approval-gate
description: The unconditional hold in front of every live send, bulk or irreversible CRM mutation, and spend change in the GTM pipeline. Classifies every action into run-freely (scoring, list building, analysis, drafting, single provenance-logged edits) versus must-gate (sequence launches and batch sends, merges and mass field writes and deletions, enrichment-credit or ad-budget changes), and holds the must-gate ones for a human yes/no. Operationalizes the repo principle "autonomous inside the job, gated at the edge." Run when asked "can this go out," "is this safe to run," "hold that send," "gate this," or before any live send, bulk CRM write, or spend change fires.
user_invocable: true
---

# live-action-approval-gate

A GTM Engineer's systems touch three things that are expensive to get wrong: live sends into real inboxes, bulk writes into the CRM, and money. Scoring a lead wrong is a cheap mistake caught on the next run. Launching a 200-lead sequence with a broken merge field, mass-updating the wrong 400 records, or burning a batch of enrichment credits on a bad list are not cheap, and not always reversible. This skill is the single hold in front of exactly those actions, and only those.

It is the operational form of the repo's fourth operating principle: autonomous inside the job, gated at the edge. Inside the job, scoring, enrichment logic, list building, drafting, and analysis all run freely. At the edge, where an action sends to a real person, mutates the system of record in bulk, or changes spend, everything stops here for a human yes/no. It holds; it does not build the list (`signal-prospecting`), draft the copy (`collateral-generator`, `reply-triage`), or run the mutation mechanics (`crm-hygiene-audit`).

## What counts as an edge action (always gated)

- **Live sends.** A sequence launch, a batch of emails or LinkedIn messages, any send that reaches real recipients. Drafting and staging them is free; releasing them is not.
- **Bulk or irreversible CRM mutations.** Merges, mass field updates, deletions, any write that changes many records or cannot be cleanly undone. Single-record edits that go through the normal provenance-logged path run freely; bulk and irreversible ones gate. This is the general form of the human-gated merge step `crm-hygiene-audit` already enforces, extended to every bulk write.
- **Spend changes.** Releasing an enrichment run that will burn credits beyond the standing ceiling, or changing an ad budget. The `spend-and-cost-guard` control decides whether a paid call is even eligible; this gate holds the human go/no-go on the spend itself.

Everything else, scoring, tiering, list assembly, dedup analysis, draft generation, forecast math, runs without gating. The gate is for the three expensive edges, not for the work that feeds them.

## Operating Rules (read first)

- **Unconditional at the edge.** No score is high enough and no list is clean enough to send, mutate, or spend without a yes. There is no "trusted enough to auto-fire" tier.
- **The named owner is the single approver.** A release comes from the human who owns that action (the rep, the RevOps owner, the growth lead in `context/people/`), never from me inferring the answer.
- **I hold, I do not act.** A yes is a signal for the send, the write, or the spend to happen through the real tool and the owner's hand. I never launch the sequence, run the merge, or spend the credits myself.
- **Building and drafting are other skills.** `signal-prospecting` and `waterfall-enrichment` build the list, `collateral-generator` and `reply-triage` draft, `crm-hygiene-audit` runs its own plan-baseline-execute-verify on a mutation. A clean, well-drafted, correctly-scored action still waits here for the go.
- **The evidence rides with the ask.** Every hold shows the action, its scope (how many records, which recipients, how much spend), the source, and one line of why, so the owner decides on the blast radius, not on trust.

## Steps

1. Take an action the pipeline has prepared (a staged send, a bulk write, a spend release).
2. Classify it: is it a live send, a bulk/irreversible CRM mutation, or a spend change? Or is it run-freely work?
3. If run-freely, let it move. If it is an edge action, hold it.
4. Build the approval ask: the action, its scope and blast radius, the linked source (the list, the dedup plan, the spend-guard result), and one line of why it is gated.
5. Present it to the owning human and hold. Nothing proceeds while it waits.
6. On **yes**, hand the release to the real tool and the owner's hand, and log the go. On **no**, keep it held and log the call. I never act on either answer myself.

## Intended Integrations (not live in this demo)

- Held sends would queue against the **sequencer** (Smartlead, Instantly, Apollo, Unipile) so the release fires the real sequence, not a simulated one.
- Held CRM mutations would carry the **HubSpot/Salesforce** merge or bulk-write payload, released only on the yes, and run through `crm-hygiene-audit`'s verify-diff after.
- Held spend releases would sit in front of the **enrichment providers** (Clay, Apollo, LeadMagic) or the **ad platform**, with `spend-and-cost-guard`'s eligibility result attached. None of it is wired here; this skill makes no live call and holds a simulated queue.

## Worked Example (fictional)

Rivergate Data, one week. Three prepared actions reach me. All three are edge actions and stop.

**1. Launch the 200-lead Series-A signal sequence (live send).** `signal-prospecting` tiered the accounts and `collateral-generator` drafted the personalized opener, validation pass clean. It is still 200 live sends into real inboxes, so it holds:

> **Launch sequence to 200 accounts?** Series-A funding signal segment, opener drafted and validated. Blast radius: 200 recipients, live. Why held: live send. Yes / No.

**2. Merge 40 duplicate company records (bulk CRM mutation).** `crm-hygiene-audit`'s dual-pass dedup found 40 duplicate companies and built the merge plan with a baseline snapshot. The merge itself is bulk and hard to reverse, so it holds:

> **Run 40 company merges?** From crm-hygiene-audit dedup pass, baseline snapshot taken. Blast radius: 40 records, irreversible. Why held: bulk CRM mutation. Yes / No.

**3. Enrich a 300-account batch (spend change).** A `waterfall-enrichment` run is staged. `spend-and-cost-guard` cleared the TIER_1/TIER_2 accounts and blocked the rest, leaving 180 eligible at roughly 180 credits. Releasing that spend gates:

> **Release enrichment spend on 180 accounts?** spend-and-cost-guard cleared 180 of 300 (TIER_1/2), ~180 credits. Blast radius: ~180 credits. Why held: spend change. Yes / No.

In all three I held. I did not tier the accounts, draft the opener, build the merge plan, or decide spend eligibility, those are `signal-prospecting`, `collateral-generator`, `crm-hygiene-audit`, and `spend-and-cost-guard`. On Marcus Iyer's (Head of Growth) yes, each release happens through the real tool; I never fire it myself.

## Adapted from

- **VERIFIED in the sibling system repos' builds:** HumanLayer (`github.com/humanlayer/humanlayer`), a purpose-built human-in-the-loop approval layer for agent actions, the closest external analogue to this hold: a specific action paused for a human yes/no before it fires. Verified in those builds, not re-fetched here, so no star figure is attached.
- **LIKELY:** the native Claude Code `PreToolUse` hook (returns an "ask" decision to force human confirmation before a tool runs) and LangChain's human-in-the-loop middleware (pauses before a side-effecting call, waits for approve/edit/reject). Documented patterns, not re-fetched this session.
- The classification of GTM's three gated edges (live sends, bulk CRM mutations, spend changes) is **my own framing**, and it is the direct operational form of this repo's fourth operating principle in `CLAUDE.md`. It composes with `crm-hygiene-audit`'s existing human-gated merge and `spend-and-cost-guard`'s eligibility check rather than duplicating them.

## Rules

- Every live send, bulk/irreversible CRM mutation, and spend change passes through me. No exceptions, at any confidence. There is no auto-fire tier.
- The owning human is the single approver.
- I hold. I do not build the list, draft the copy, run the mutation mechanics, or decide spend eligibility. A clean, correct action still waits for the go.
- A yes routes to the real tool and the owner's hand. I never send, merge, or spend myself.
- The ask carries its blast radius: how many recipients, how many records, how much spend, plus source and one line of why.
