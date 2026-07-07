---
name: spend-and-cost-guard
description: The cross-cutting cost control that sits underneath every paid GTM step (enrichment credits, deep-research and LLM-at-scale for collateral, paid data providers), not a pipeline stage of its own. Gates a paid call behind a cheap deterministic ICP-tier pre-check, checks a freshness cache before re-spending on an account already processed, rotates across provisioned provider keys on a rate-limit instead of stalling, and tracks cumulative spend per account so cost-per-qualified-lead stays proportional to fit instead of uniform. waterfall-enrichment's inline spend gate is one instance of this control; this is the always-on version under every paid step. Run before any paid enrichment, research, or generation call, or when asked to check the spend gate, check the cache, or explain why an account was blocked from a paid call.
user_invocable: true
---

# spend-and-cost-guard

Every interesting GTM step past qualification costs real money: enrichment credits, deep-research calls, LLM generation at scale for personalized collateral, paid data providers. The failure mode is not overspending in total, a budget can look fine for the month and still be burning evenly across good-fit and bad-fit accounts. The failure mode is spending *disproportionately*: paying the same to research a TIER_4 account as a TIER_1 one. This control keeps spend proportional to fit, and makes "are we spending proportionally" a number that can be checked rather than a guess.

It is a cross-cutting control, not a stage. `waterfall-enrichment` already runs an ICP-gate-before-spend for its own enrichment flow; that is one instance of this pattern. This skill is the always-on version that sits under *every* paid step, with four mechanics in order: a cheap deterministic gate before a paid call, a freshness-cache check before a re-spend, provider-key rotation when a limit is hit mid-run, and a running per-account spend ledger. It decides whether a paid call is *eligible*; it does not hold the human go/no-go on a spend change (that is `live-action-approval-gate`) and it does not run the enrichment itself (that is `waterfall-enrichment`).

## Operating Rules (read first)

- **The gate is a hard stop, not a soft signal.** A TIER_3/TIER_4 account does not get a paid call "just this once." A spend gate that can be waived is not a gate.
- **This skill is a checkpoint, not the doer.** It decides whether a paid step is allowed to run and logs what happened; it does not make the provider call itself.
- **Cache reuse is bounded by the freshness window.** A cache hit inside the window is a legitimate zero-cost result. A cache entry past the window, or invalidated by a real change in the account's signals, is a hard miss and goes through the full gated path, never a quiet stale serve.
- **Key rotation is a capacity fallback, not a policy workaround.** Rotating across provisioned keys to keep a qualified account moving is fine. Rotating to dodge a provider's actual rate limit or terms is not, and this skill does not do the latter. If the whole pool is exhausted, it stops and flags for a human, a visible honest state.
- **Every credit, spent or not, is attributed to a specific account.** A cost figure with no account attached can only answer "is the month's total fine," not "is spend proportional to fit," which is the question that matters.

## Steps

1. **Load the account's tier and cache state.** Read the `icp_tier` set upstream by `lead-routing` / the ICP score; this skill does not recompute fit. Read the freshness cache for this account and this specific paid step (a research cache and an enrichment cache are separate entries), and note its age against the freshness window.
2. **Run the spend gate (hard, before any paid call).** TIER_1 and TIER_2 clear; TIER_3 and TIER_4 do not. On a no-match, stop, log the account as blocked-pre-spend with the reason, and touch no provider. On a match, proceed.
3. **Check the cache before a fresh spend.** A fresh cache hit inside the window skips the paid call and is logged as a zero-cost hit. No entry, or an aged/invalidated one, is a hard miss and proceeds to a fresh call. A material change in the account's signals invalidates the cache regardless of age.
4. **Rotate keys on a limit, never bypass.** Once a call is allowed, it uses the next provisioned key. On a rate-limit or credit-ceiling response mid-run, rotate to the next key and retry, logging the rotation. If the whole pool is exhausted, stop and flag for a human decision; do not loop looking for a workaround.
5. **Log spend and update the per-account ledger.** Every attempt (hit, miss, cache hit, rotation, blocked gate) is logged with its cost against the account, and rolled into a running `cumulative_spend`, so cost can be compared against `icp_tier` across a batch to catch a low-fit account that slipped the gate or a high-fit one absorbing far more than its research needed.

## Intended Integrations (not live in this demo)

- **The paid providers** this gate protects spend on: enrichment/contact-data APIs (Clay, Apollo, LeadMagic, ZoomInfo) and any LLM-based deep-research or collateral-generation step. In a live build this skill's gate result is the precondition those calls run behind, not a call this skill makes.
- **A keyed cache store** (Redis or similar) would back the freshness cache with a real per-step TTL.
- **A secrets manager / key vault** (AWS Secrets Manager, Doppler) would hold the provisioned key pool and report real quota/429 state, instead of this skill inferring it.
- **The CRM / data warehouse** would carry the real `cumulative_spend` field and feed a usage dashboard that answers the cost-per-tier proportionality question. None of it is wired here.

## Worked Example (fictional)

Rivergate Data, three accounts run through the gate in order. Marcus Iyer (Head of Growth) owns the enrichment budget line.

**Northwind Labs, TIER_1, no cache.** Gate: TIER_1 clears, match. Cache: no entry, fresh miss, proceed. Call: the first provisioned key returns a rate-limit partway through, rotate to the second key, call completes. Ledger: `result: hit, rotation: 1, cost: 1 credit`, `cumulative_spend: 1`.

**Delta Ops, TIER_2, cached 5 days ago.** Gate: TIER_2 clears, match. Cache: 5 days against a 14-day window, fresh hit, paid call skipped. Ledger: `result: cache_hit, cost: 0`, `cumulative_spend` unchanged. The account was enriched last week and nothing material changed, so re-spending would buy nothing.

**Some Low-Fit Co, TIER_4.** Gate: TIER_4 does not clear, no-match. Stop. No provider is called; steps 3 and 4 never run. Ledger: `result: blocked_pre_spend, cost: 0`.

Across the batch: one credit spent, on the one TIER_1 account that needed a fresh call. The TIER_2 account cost nothing because its data was current; the TIER_4 account cost nothing because it never should have been spent on. That is the proportionality the control exists to produce, and the per-account ledger is what lets Marcus see it, not just a month-total that looks fine.

## Adapted from

- **The gate: `wreynoir/company-enrichment-skill`** (`github.com/wreynoir/company-enrichment-skill`), whose Phase 1 runs a cheap check against a stored ICP file and prints an explicit MATCH / NO-MATCH before any paid Phase 2 call. This is the **same real repo cited for `waterfall-enrichment`'s inline gate** in this repo; reused here for the broader always-on control rather than one enrichment step. Verified in the sibling builds; treat any star count as approximate and stale.
- **The cache:** a generic cache-aside pattern (deterministic key from method/params, per-endpoint TTL, invalidation on change), adapted from a general-purpose API-caching skill, not one built for enrichment. Worth saying plainly: this is a general pattern applied to this use case, and it deliberately drops stale-while-revalidate, an aged research record served as current could produce a wrong outreach angle, so an aged entry is a hard miss, not a background refresh.
- **Key rotation: designed from a real pattern, not ported.** No genuine Claude Code key-rotation skill was found in the sibling search (the one candidate was hollow boilerplate); the real reference was a maintained multi-account proxy implementing quota-based rotation and 429 handling. Step 4 is designed from that pattern.
- **Per-account cumulative spend tracking: my own design.** Not found as a citable mechanic; built to satisfy the "keep cost proportional to fit" requirement directly.

## Rules

- The step-2 gate is a hard stop on TIER_3/TIER_4 accounts. No "run it anyway" path.
- A cache hit inside the freshness window skips the paid call. An aged or signal-invalidated entry is a hard miss, not a stale serve.
- Rotate only to the next provisioned key on a limit. If the pool is exhausted, stop and flag for a human; never invent a workaround.
- Every attempt (hit, miss, cache hit, blocked gate) is logged with its cost against the account. A miss still cost a credit in most pricing models; log it as a miss.
- This skill decides eligibility. It does not hold the human spend go/no-go (`live-action-approval-gate`) or run the enrichment (`waterfall-enrichment`).
