---
name: waterfall-enrichment
description: Two-phase account enrichment with a spend gate. Phase 1 checks an account against the stored ICP definition for free, binary MATCH/no-match, halting before any paid lookup on no-match. Phase 2 (qualified accounts only) runs a waterfall across enrichment providers in priority order for email, direct-dial phone, and firmographic/technographic data, one provider at a time until a hit, then dedupes against the CRM before writing. Logs credit spend per lead so cost-per-qualified-lead stays trackable. Run on any account surfaced by signal-prospecting before it goes to outreach.
user_invocable: true
---

# Waterfall Enrichment

Turns a raw account or contact into an enrichment-complete CRM record without burning provider credits on accounts that were never going to qualify. The core discipline is ordering: free checks first, cheapest paid checks next, and stop the instant a good-enough answer is found. Enrichment is a cost center, not a checkbox — this skill treats it that way at every step.

## Adapted from

This skill's structure is adapted from two open patterns, not invented from scratch:

- **wreynoir/company-enrichment-skill** (github.com/wreynoir/company-enrichment-skill) — the 3-phase "check ICP fit before spending" gate, and per-lead credit-spend logging so cost-per-qualified-lead is a tracked number, not a guess.
- **getaero-io/gtm-eng-skills** (github.com/getaero-io/gtm-eng-skills), specifically the `build-tam` skill — sourcing accounts from multiple providers, then deduping and merging before anything gets written, and the Clay-style waterfall provider-chain concept (try provider A, fall through to B, then C, don't fan out to all three at once).

## Operating Rules (read first)

- **Phase 1 is a hard gate, not a soft signal.** No-match on the stored ICP definition means STOP. Do not run Phase 2 "just to see." The entire point of a two-phase design is that Phase 2 costs money and Phase 1 doesn't.
- **Phase 1 is binary, not a score.** MATCH or no-match against explicit fit criteria (industry, headcount band, tech stack signal, funding stage, geography). Don't produce a numeric ICP score here — that invites borderline accounts to sneak through on a "70/100 is close enough" rationalization. If a criterion is ambiguous, that's a MATCH with a caveat noted, not a fractional score.
- **Waterfall means sequential, not parallel.** Within each data type (email, phone, firmographic), try providers in priority order and stop at the first hit. Never query all providers for the same field on the same lead — that's paying 3-4x for one answer.
- **Dedup before write, always.** Every enriched record gets checked against the existing CRM/warehouse (by domain + normalized name, then by email if domain match is ambiguous) before a new record is written. A near-duplicate gets merged into the existing record, not inserted alongside it.
- **Every spent credit gets logged against the lead it was spent on.** No exceptions, including a provider miss (a miss still cost a credit or an API call in most pricing models — log it as a miss, not silently).
- **Every field in the output record carries its source provider.** This isn't optional metadata — it's how a future data-quality problem gets traced back to which vendor's data went stale or was wrong.
- **This skill enriches; it doesn't qualify from scratch.** It assumes `signal-prospecting` already found the account and handed over a reason it's on the radar. Phase 1 here is a fit re-check, not the original discovery.

## Intended Integrations (not live in this demo)

- **Clay** would be the waterfall orchestration layer itself — sequenced provider calls, built-in dedup logic, and a native credit-spend ledger per row. In a live build, most of Phase 2's mechanics live inside a Clay table, not in this skill's own logic.
- **Apollo** — first-priority provider for company + contact-level firmographic data and a solid secondary source for business email.
- **Hunter.io** — first-priority provider for verified business email on a known contact (domain-search + email-finder pattern), tried before ZoomInfo because it's materially cheaper per successful match.
- **ZoomInfo** — fallback provider for email and firmographic/technographic data when Apollo/Hunter miss, and primary source for org-chart and intent-signal data where budget allows.
- **LeadMagic** — priority provider for direct-dial and mobile phone numbers, tried before ZoomInfo's phone data on cost grounds.
- **The data warehouse / CRM (Snowflake, HubSpot, or Salesforce)** would be the dedup source of truth in Step 3, queried live instead of the placeholder lookup used in this demo.

None of these are wired in this repo. This is a demonstration of the operating logic, provider ordering, and gating discipline, not a live pipeline. The provider order below reflects realistic per-field cost, not a specific negotiated contract.

## Step 1: Load the ICP Definition and the Incoming Account

Read the stored ICP definition (fit criteria: industry, headcount band, funding stage, tech-stack signals, geography, any explicit disqualifiers). Read the incoming account record, which should already carry the reason `signal-prospecting` flagged it (a trigger event, a signal source, a lookalike match).

If the account has no clear source or reason attached, that's a process gap worth flagging back to signal-prospecting, not something to paper over by inventing a justification here.

## Step 2: Phase 1 — Free ICP Fit Check (no spend)

Compare the account against each stored fit criterion using only what's already on hand (the account record, public firmographic facts already known, the signal that surfaced it). No provider is called yet.

- **MATCH**: every hard criterion is satisfied (or satisfied with a noted caveat on an ambiguous one). Proceed to Phase 2.
- **NO-MATCH**: any hard criterion fails (wrong industry, headcount outside band, disqualified geography). **STOP HERE.** Log the account as disqualified with the specific failing criterion, and do not touch Phase 2 for it. This is the entire cost-control point of the skill — say it plainly in the output, don't bury a "did not qualify" account inside a partial enrichment record.

Report the gate result before doing anything else. A no-match account gets a one-line disposition, not a partial write.

## Step 3: Phase 2 — Waterfall Enrichment (qualified accounts only)

For each data type below, try providers in the stated priority order and stop at the first successful hit. Log every attempt (hit or miss) with its provider and the credit/cost it consumed.

**Email (target: 85%+ coverage across qualified accounts)**
1. Hunter.io domain-search + email-finder on the known contact name + company domain
2. Apollo contact-level email lookup
3. ZoomInfo email lookup (fallback, higher cost per record)

**Phone / direct-dial**
1. LeadMagic direct-dial lookup
2. ZoomInfo phone data (fallback)

**Firmographic / technographic**
1. Apollo company profile (headcount, industry, funding stage, tech stack detected)
2. ZoomInfo org-chart and technographic data (fallback, and primary source for intent signals where in scope)

If a data type exhausts its full provider list with no hit, mark that field `unresolved` rather than leaving it blank with no explanation — an unresolved field and a field nobody tried to fill are different data-quality states and should read differently downstream.

## Step 4: Dedup Against CRM/Warehouse

Before writing anything, check the enriched account + contact against the existing CRM/warehouse:

1. Match on normalized company domain first (strip protocol, www, trailing slash).
2. If domain match is ambiguous (subsidiary, rebrand, multiple domains on file), fall back to normalized company name + geography.
3. For contact-level records, match on verified email once resolved; if no email yet, match on full name + company.

- **Exact match found**: merge the newly enriched fields into the existing record. Never create a duplicate row. Prefer the newer data for any field that conflicts, but keep the old value in a note if the conflict is material (e.g., headcount jumped from 40 to 200 — that's a signal worth keeping, not silently overwriting).
- **No match found**: this is a new record, proceed to write.
- **Partial/uncertain match**: flag for manual review rather than guessing. Don't auto-merge on a fuzzy name match alone.

## Step 5: Write the Enriched Record and Spend Log

Output two things per processed account:

**1. Enriched record** (structured schema):

```
account:
  company_name:
  domain:
  icp_gate: MATCH | NO-MATCH
  icp_gate_notes: [failing criterion, or caveat on an ambiguous MATCH]
  industry:
  headcount_band:
  funding_stage:
  tech_stack_signals: []
  source_signal: [what signal-prospecting flagged, and when]

contact:
  name:
  title:
  email:
    value:
    provider: [Hunter | Apollo | ZoomInfo | unresolved]
  phone:
    value:
    provider: [LeadMagic | ZoomInfo | unresolved]

firmographic:
  provider: [Apollo | ZoomInfo]
  headcount:
  industry:
  funding_stage:
technographic:
  provider: [Apollo | ZoomInfo | unresolved]
  detected_stack: []

dedup_result: new_record | merged_into_existing | flagged_for_review
dedup_match_id: [CRM record ID if merged]
```

**2. Spend log entry** (per lead, per attempt):

```
lead: [company_name]
attempts:
  - field: email        provider: Hunter.io    result: hit    cost: [credit/API unit]
  - field: phone        provider: LeadMagic    result: miss   cost: [credit/API unit]
  - field: phone        provider: ZoomInfo     result: hit    cost: [credit/API unit]
  - field: firmographic provider: Apollo       result: hit    cost: [credit/API unit]
total_credits_spent: [sum]
qualified: true
```

Roll this up across a batch into a simple **cost-per-qualified-lead** number (total credits spent on qualified accounts ÷ number of qualified accounts) so the waterfall's actual efficiency is visible, not assumed.

## Worked Example (Rivergate Data)

`signal-prospecting` flags **Northbeam Robotics** (Series A, industrial automation, 65 employees) off a funding-announcement signal, and hands it to this skill.

**Step 2 — Phase 1 gate:**
ICP definition requires B2B SaaS or infra-adjacent, Series A-B, 30-250 employees, US/UK/EU. Northbeam is industrial automation with a software control layer, 65 employees, Series A, US-based. Industry criterion is a caveat (hardware-adjacent, not pure SaaS) but the account sells a SaaS control platform to its customers, satisfying the underlying "sells recurring software" intent. **MATCH, with caveat: hardware-adjacent, verify software-revenue mix before high-touch outreach.**

**Step 3 — waterfall:**
- Email: Hunter.io hit on the VP Ops contact — 1 credit, done.
- Phone: LeadMagic miss (no direct-dial on file), ZoomInfo hit — 2 attempts, 1 credit + 1 credit.
- Firmographic: Apollo hit (68 employees, confirms headcount band, detected AWS + Salesforce stack) — 1 credit.

**Step 4 — dedup:**
Domain `northbeamrobotics.com` not found in the CRM. New record, proceed to write.

**Step 5 — output:**
Enriched record written with `email.provider: Hunter.io`, `phone.provider: ZoomInfo`, `firmographic.provider: Apollo`. Spend log: 4 credits total, `dedup_result: new_record`, `qualified: true`. Rolled into the week's cost-per-qualified-lead figure alongside the batch's other accounts.

Compare against a fictional same-day account, **Larchmont Legal Services** (200-person law firm), which fails the ICP gate outright on industry. Phase 1 logs `NO-MATCH — industry: legal services, not SaaS/infra-adjacent`, and Phase 2 never runs. Zero credits spent.

## Rules

- Phase 1 no-match always halts before any provider is called. No exceptions, no "quick check anyway."
- Phase 1 is MATCH/no-match, never a numeric score.
- Never call more than one provider per field per lead unless the first attempt was a miss.
- Every enriched field carries its source provider; every unresolved field says so explicitly instead of being left blank.
- Dedup runs before every write, matching on domain first, then name+geography, then email/name for contacts. Uncertain matches go to manual review, not an auto-merge.
- Every provider attempt, hit or miss, gets logged with its cost. The spend log is what makes cost-per-qualified-lead a real, trackable number instead of a guess.
