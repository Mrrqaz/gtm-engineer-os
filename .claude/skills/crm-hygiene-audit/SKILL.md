---
name: crm-hygiene-audit
description: Weighted A-F CRM/pipeline health score across five dimensions (Pipeline Health, Data Quality, Forecast Hygiene, Deal Integrity, Process Integrity), plus a dual-pass company/contact deduplication pass and orphaned-record cleanup. Every action runs Plan then Baseline-snapshot then Execute then Verify-diff, so nothing lands without a before/after comparison. Merges and other irreversible actions stay human-gated. Run this weekly, before a forecast call, or whenever a rep or VP Sales flags "the CRM data can't be trusted."
user_invocable: true
---

# CRM Hygiene Audit

CRM data doesn't rot randomly. It rots in the same handful of places every time: stale opportunities nobody touches, late-stage deals with no next step, duplicate company records from two reps prospecting the same account, and reps hand-typing data that enrichment already delivered. This skill scores the mess, proposes fixes, and never executes an irreversible change without a human looking at it first.

## Adapted from

This skill's mechanics aren't invented from scratch. Two open-source skill packs are the direct source, credited here rather than silently reused:

- **elijeangilles/revops-skills** (`salesforce-revops-audit`) — the weighted, 100-point-baseline scoring formula across five named dimensions is adapted from this repo's audit skill.
- **TomGranot/hubspot-admin-skills** (32 skills) — the dual-pass dedup mechanic (from `merge-duplicate-companies`), the Plan → Baseline → Execute → Verify-diff scaffold used across that whole repo, and the orphaned-record cleanup pattern are adapted from here.

## Operating Rules (read first)

- **Never execute a merge, a delete, or a reassignment without a human approving the plan first.** This skill drafts; it does not act unilaterally on anything that can't be trivially undone.
- **Every hygiene action gets a baseline snapshot before it runs.** No exceptions, even for "obviously safe" changes like reassigning an orphaned contact. If there's no before-state, there's no diff, and if there's no diff, there's no way to prove the fix worked or catch a mistake.
- **Score the account as it is, not as it should be.** Don't round a 62 up to a 70 because the rep "usually" keeps things tidy. The formula is mechanical on purpose so two audits of the same data land on the same score.
- **Fuzzy-match dedup candidates are a lead, not a verdict.** Pass 2 (name-based) results always get a confidence flag and route to human review before Pass 1 (domain-based) treatment.
- **A deduction needs a named, countable trigger.** If a point deduction can't be traced to a specific query result (a count, a percentage, a date delta), it doesn't belong in the score.

## Intended Integrations (not live in this demo)

- **HubSpot API** (or **Salesforce API / SOQL**) would power every pull in this skill: open opportunities, activity timestamps, stage/probability pairs, owner status, company/contact records for dedup. Right now this skill reads whatever CRM export or CSV the user provides.
- **HubSpot Workflows API** or **Salesforce flow/automation** would eventually auto-flag orphaned records the moment an owner is deactivated, instead of catching it on the next scheduled audit.
- **Clay** enrichment webhook would auto-populate the fields reps currently re-key by hand (see Dana Whitfield's flag below) — the fix for that specific finding lives outside this skill, but this skill is what surfaces the finding.
- None of this is wired here. Every step below runs on a pasted or attached CSV export, and says so plainly when no data has been provided.

## Step 1: Establish the Baseline Snapshot

Before touching anything, export or accept the current state as a CSV (or ask the user to paste/attach one): open opportunities with stage, probability, owner, last-activity date, next-step field, and amount; company and contact records with domain, name, and owner.

Save this as the baseline. Every later diff in Step 5 compares against this file, not against memory of what "used to be there." If no data is provided, say so and stop, don't audit against assumptions.

## Step 2: Score the Five Dimensions

Each dimension starts at **100** and takes named deductions. Final dimension score floors at 0 (deductions don't go negative). Deductions are additive within a dimension — if three triggers fire, all three deductions apply.

### Pipeline Health (weight: 25%)
- **-20** if more than 25% of open opportunities have no logged activity in 14+ days
- **-15** if more than 15% of open opportunities have sat in the same stage for 45+ days with no stage change
- **-10** if the pipeline has fewer than 3x coverage against the current-quarter quota number (a coverage gap, not a data-entry problem, but it degrades the same score because it's what the number is *for*)
- **-10** if more than 10% of opportunities are missing an amount field entirely

### Data Quality (weight: 20%)
- **-20** if more than 20% of contact records are missing both email and phone
- **-15** if more than 15% of company records show a manually-typed field that duplicates a known enrichment field (title, company size, industry) with a stale or conflicting value against the enrichment source
- **-10** for every enrichment-sourced field found hand-overwritten with a lower-quality value in the last 30 days (capped at -20 total under this trigger)
- **-10** if more than 10% of records have obviously malformed data (placeholder emails, "N/A" in a required field, test records left live)

### Forecast Hygiene (weight: 25%)
- **-25** if more than 10% of "Commit" or "Best Case" tier deals have no activity in the last 7 days
- **-15** if late-stage deals (final two stages before Closed Won) lack a populated next-step field
- **-15** for probability-to-stage mismatch greater than 25 points (e.g., a deal sitting at a stage whose default probability is 40% but manually set to 85%, with no note explaining why)
- **-10** if any closed-lost deal is missing a loss-reason field

### Deal Integrity (weight: 20%)
- **-20** if more than 5% of open deals have a close date more than 60 days in the past (a dead giveaway of an un-worked pipeline nobody's cleaning up)
- **-15** if duplicate open opportunities exist against the same company with overlapping amounts (a sign of two reps working the same account without coordination)
- **-15** if more than 10% of opportunities are missing a linked primary contact
- **-10** if deal-stage naming is inconsistent across the pipeline (custom stages added ad hoc that don't map to the standard funnel)

### Process Integrity (weight: 10%)
- **-30** if there is no standard lead-routing SLA being tracked, or the SLA has no owner
- **-25** if required fields (source, stage, next step) can be left blank at creation with no validation
- **-20** if more than 15% of records show no owner assigned at all (not just a deactivated owner, genuinely unowned from creation)
- **-15** if there's no audit trail (no field history / no changed-by log) on stage or amount changes

### Composite Score

```
Composite = (Pipeline Health x 0.25) + (Data Quality x 0.20)
          + (Forecast Hygiene x 0.25) + (Deal Integrity x 0.20)
          + (Process Integrity x 0.10)
```

Grade the composite: **A** 90-100, **B** 80-89, **C** 70-79, **D** 60-69, **F** below 60. Report the composite grade plus each dimension's own score and its specific deductions, never just the final letter, the deductions are the action list.

## Step 3: Run the Dual-Pass Dedup

**Pass 1 — Normalized-domain exact match (high confidence).** Strip protocol, `www.`, and trailing slash from every company record's domain field, lowercase it, and match exactly. Two company records with the same normalized domain are treated as a confirmed duplicate pair, no human judgment needed to identify them, though the merge itself still needs approval (Step 4).

**Pass 2 — Normalized-name fuzzy match (higher false-positive risk).** For records that didn't match on domain, normalize company names (strip legal suffixes like Inc/LLC/Ltd, lowercase, strip punctuation) and run a similarity match. Flag pairs above the similarity threshold as **candidates only**. This pass catches real duplicates that Pass 1 misses (a company with two different domains, a rebrand, a typo'd domain) but also throws false positives (two genuinely different companies with similar generic names). Every Pass 2 result is labeled `REVIEW REQUIRED — name-match, not domain-confirmed` and never auto-queued for merge the way a Pass 1 match can be.

Contact-record dedup follows the same two passes: Pass 1 on normalized email domain + exact email match, Pass 2 on normalized full name within the same company record.

## Step 4: Draft the Action Plan (human-gated)

Compile everything from Steps 2 and 3 into one plan, split into two lists:

**Auto-executable on approval** (reversible, low-risk, still requires a yes before running):
- Reassigning records currently owned by a deactivated user to a stated fallback owner
- Flagging (not deleting) unowned contacts for owner assignment
- Backfilling a missing next-step field with a prompt to the deal owner, not a synthetic value

**Requires explicit human sign-off before any execution, one at a time:**
- Every proposed merge (Pass 1 and Pass 2 both), with the two source records and the proposed surviving record shown side by side
- Any deletion
- Any bulk field overwrite

Never batch-approve a list of merges with one "yes." Each merge plan is its own line item so a human can reject one without blocking the rest.

## Step 5: Execute Approved Actions, Then Verify

For everything approved in Step 4: execute, then re-pull the affected records and diff them against the Step 1 baseline. The verification output is always a **before/after diff**, not a "done" confirmation. If a merge was approved, show the surviving record's field values next to what both source records had, so a wrong field-precedence choice (picking the stale value over the fresh one) is catchable immediately, not three weeks later.

If any diff doesn't match what the plan said it would do, stop and flag it before touching anything else. Don't proceed down the action list assuming the mismatch was a fluke.

## Worked Example (Rivergate Data, fictional)

Dana Whitfield (VP Sales) flagged that reps are manually re-entering data Clay already enriched, and separately wants at-risk deals surfaced before the Friday forecast call. Both show up in this audit without needing a separate ask.

Baseline CSV: 340 open opportunities, 890 company records, 1,240 contact records.

```
CRM Hygiene Audit — Rivergate Data — 2026-07-02

COMPOSITE: 69 / 100 (D)

Pipeline Health (weight 25%)
  -20  29% of open opps have no activity in 14+ days
  -15  17% of open opps stuck in same stage 45+ days
  0    coverage is 3.4x quota, no deduction
  -10  11% of opps missing an amount field
  → 100 - 45 = 55

Data Quality (weight 20%)
  -20  24% of contacts missing both email and phone
  -15  18% of company records show a hand-typed field conflicting with a
       known Clay-enriched value — this is Dana's exact complaint,
       confirmed in the data, not just anecdotal
  -10  Clay-enriched "company size" field hand-overwritten on 14 records
       in the last 30 days (capped deduction)
  -10  N/A
  → 100 - 45 = 55

Forecast Hygiene (weight 25%)
  -25  N/A, Commit/Best Case activity is current
  -15  Late-stage deals (Negotiation, Verbal) missing next-step field: 22%
       of that population — this is what "at-risk before Friday" means
       concretely
  -15  N/A
  -10  N/A
  → 100 - 15 = 85

Deal Integrity (weight 20%)
  -20  N/A
  -15  6 duplicate open opportunities found against the same company
       (matches Pass 1 dedup output below)
  0    N/A
  -10  N/A
  → 100 - 15 = 85

Process Integrity (weight 10%)
  -30  N/A, SLA exists (5-minute inbound-hot window, approved 2026-06-23)
  -25  Required fields can be left blank at creation, confirmed
  -20  N/A
  -15  No field-history audit trail on amount changes
  → 100 - 40 = 60

COMPOSITE = (55 x .25) + (55 x .20) + (85 x .25) + (85 x .20) + (60 x .10)
          = 13.75 + 11 + 21.25 + 17 + 6 = 69.0 → grade D

DEDUP RESULTS
Pass 1 (domain-exact): 6 confirmed duplicate company pairs, all in
  the same 6 accounts driving the Deal Integrity deduction above.
Pass 2 (name-fuzzy): 11 candidate pairs flagged REVIEW REQUIRED. Two are
  almost certainly the same company with a rebranded domain
  ("Rivergate Analytics" / "Rivergate Data Labs"); the other nine look
  like coincidental name overlap between unrelated companies and should
  likely be rejected on review.

ORPHANED RECORDS
- 4 opportunities and 19 contacts owned by a rep deactivated 2026-06-10,
  never reassigned. Proposed fallback owner: current AE team lead,
  pending approval.
- 31 contacts with no owner field populated at all (not deactivation,
  never assigned).

ACTION PLAN
Auto-executable on approval:
  1. Reassign the 4 opps + 19 contacts from the deactivated rep to the
     stated fallback owner.
  2. Flag the 31 unowned contacts for manual owner assignment (list
     attached, not auto-assigned).

Requires individual sign-off:
  1. Merge company pair A (Pass 1, domain-confirmed) — surviving record
     shown with field-by-field precedence.
  2-6. [Merge company pairs B-F, same format]
  7. Review "Rivergate Analytics" / "Rivergate Data Labs" (Pass 2,
     name-only) before treating as a merge candidate.

Dana's specific complaint (hand-typed fields overwriting Clay enrichment)
is confirmed at 18% of company records. Recommend routing this to the
Clay-to-HubSpot auto-populate fix rather than a one-time cleanup, or this
score regresses again within a month.
```

## Rules

- Never merge, delete, or bulk-overwrite without an explicit per-item human approval. A list of ten proposed merges is ten approvals, not one.
- Never score against a target or a "should be" state. The score is what the baseline CSV actually shows.
- A Pass 2 (name-fuzzy) dedup candidate is never treated with the same confidence as a Pass 1 (domain-exact) match. Label them differently every time they appear.
- No action runs without a prior baseline snapshot and a post-action diff. If either is missing, the action didn't happen as far as this skill is concerned.
- If a deduction can't be tied to a specific, countable trigger in the data, leave it out rather than estimate it.
- If no CRM export has been provided, say so and stop. Don't audit from memory of a prior run.
