---
name: competitive-intel-monitor
description: Multi-verb competitive intelligence system tracking 8 dimensions per competitor (product, pricing, funding, hiring, partnerships, wins against us, losses against us, messaging/positioning) across three cadences (monthly tier-1 review, event-triggered 48-hour response window, quarterly full landscape reassessment). Five commands - /ci:landscape (full market map), /ci:battlecard [competitor] (sales-facing battlecard), /ci:winloss (competitive reasoning from won/lost deals), /ci:update [competitor] (single-competitor refresh), /ci:map (visual/tabular overview). After the first full scan of a competitor, every repeat run is diff-only - a ~300-word delta against the prior stored brief, scored red/yellow/green. Every claim requires a cited source (website, reviews, job postings, minimum 3 source types). Run monthly for tier-1 competitors, immediately on a major competitor move (funding, pricing, launch), and quarterly for the full landscape.
user_invocable: true
---

# Competitive Intel Monitor

A monitor, not a report generator. The failure mode this skill is built against is the one Rivergate actually had: a competitor quietly cut price 15%, nobody was watching the pricing dimension between monthly reviews, and the team found out from a lost deal instead of from the monitor. A report generator that re-summarizes the whole market every time it's asked isn't a monitor, it's a research assistant with amnesia. This skill's job is to remember what it saw last time and only speak up about what changed, with a severity attached, fast enough to matter.

## Adapted from

This skill's structure is adapted from three open patterns, not invented from scratch:

- **alirezarezvani/claude-skills** (github.com/alirezarezvani/claude-skills), `c-level-advisor/skills/competitive-intel` — the multi-verb command structure and the 8-dimension / 3-cadence tracking model.
- **mohitagw15856/pm-claude-skills**, `competitive-intelligence-monitor` — the diff-only recurring-monitor mechanic: store the prior brief, compare, output only the delta, and score it red/yellow/green.
- **mohitagw15856/pm-claude-skills**, `product-team/competitive-teardown` — the evidence-citation requirement: every claim (there, a SWOT bullet; here, a battlecard line) must cite a specific data point, minimum 3 source types per competitor.

## Operating Rules (read first)

- **First run on a competitor is a full scan. Every run after that is diff-only.** Check for a stored prior brief before doing anything else. If one exists, the output is the delta, not a re-dump of all 8 dimensions. This is the entire point of the skill — skipping it turns a monitor back into a report generator.
- **Every claim needs a cited source.** A pricing claim cites a pricing-page screenshot with a date. A positioning claim cites specific website copy. A hiring claim cites a job posting URL or title. A win/loss claim cites the deal it came from. No source, no claim — flag it as unverified instead of stating it as fact.
- **Minimum 3 source types per competitor, no exceptions.** At minimum: website/pricing page, review site (G2/Capterra), job postings. A competitor profile built on one source type is incomplete, say so rather than filling gaps with inference.
- **Pricing is event-triggered, not just monthly.** This is a direct response to the Marcus Iyer incident (see Worked Example) — a competitor cut price 15% between monthly reviews and Rivergate only found out from a lost deal. Pricing-page changes, funding rounds, and major launches all get the 48-hour response window regardless of where they fall in the monthly/quarterly cycle.
- **Red/yellow/green is a severity call, not a mood.** Red = actively costing deals or requires a same-week response (a pricing cut, a feature parity gap surfaced in a live deal). Yellow = worth tracking, no immediate action (a hire in an adjacent function, a minor messaging tweak). Green = noted, no action implied (a blog post, a minor UI update). Don't inflate everything to red to seem thorough, and don't downgrade a real pricing move to yellow because it's inconvenient to act on.
- **Win/loss reasoning comes from deal data, not assumption.** `/ci:winloss` pulls from actual CRM-logged win/loss reasons tied to a named competitor. If no deal data exists for a competitor, say so — don't back-fill a plausible-sounding narrative.
- **A battlecard is a sales tool, not an intelligence dump.** `/ci:battlecard` compresses the 8-dimension profile into what a rep needs mid-call: where we win, where we lose, how to handle their pitch, and the sourced facts behind each claim. Keep it skimmable.

## Intended Integrations (not live in this demo)

- **A web-monitoring/change-detection service** (e.g. Visualping, Distill, or a scraping job scheduled through a workflow tool) would watch each tracked competitor's pricing page, homepage, and changelog on a schedule, firing the event-triggered 48-hour window automatically instead of relying on someone noticing manually. This is precisely the gap that caused the Marcus Iyer pricing-cut miss.
- **G2 / Capterra APIs** (or their public review pages, scraped on a schedule) would power the review-quote source type — new reviews, rating shifts, and recurring complaint themes feeding directly into the wins/losses and messaging dimensions.
- **A job-postings monitor** (e.g. a scraper against the competitor's careers page, or a service like Datanyze/Repvue-style tracking) would power the hiring dimension — new senior hires, headcount shifts by function, and postings that hint at roadmap direction (e.g. a sudden run of "enterprise" titled roles signals an upmarket push).
- **CRM win/loss fields** (HubSpot/Salesforce closed-lost reason + competitor-mentioned field) would power `/ci:winloss` directly instead of requiring pasted deal notes.
- None of these are wired into this repo. This is a demonstration of the tracking model, the diff mechanic, and the citation discipline — not a live monitoring pipeline. Every command below runs on whatever source material is pasted in or already stored from a prior run, and says so plainly when material is missing.

## The 8 Tracked Dimensions

Every competitor profile, full or diffed, is organized against these 8:

1. **Product** — features shipped, features deprecated, roadmap signals
2. **Pricing** — list price, packaging/tiers, discounting behavior, pricing-page structure
3. **Funding** — rounds raised, investors, valuation signals, burn/runway inference where public
4. **Hiring** — headcount trend, senior hires, function-level hiring patterns (roadmap tells)
5. **Partnerships** — integrations announced, channel/reseller deals, co-marketing
6. **Wins (against us)** — deals we lost to them, with reason if known
7. **Losses (against us)** — deals they lost to us, with reason if known
8. **Messaging/positioning** — homepage copy, tagline changes, category claims, target-segment shifts

## The Three Cadences

| Cadence | Trigger | Scope |
|---|---|---|
| **Monthly tier-1 review** | Calendar (first week of month) | Full 8-dimension check on tier-1 competitors only. Diff-only after first run. |
| **Event-triggered 48-hour window** | A major move is flagged: funding round, pricing change, major product launch | Single competitor, focused on the affected dimension(s), same diff/severity treatment, compressed timeline |
| **Quarterly full landscape reassessment** | Calendar (start of quarter) | All tracked competitors (not just tier-1), plus a check for new entrants not yet on the list |

Tier-1 vs tier-2 is a stored classification (direct competitors seen in live deals) not a judgment call made fresh each run — read it from the stored competitor list rather than re-deciding it every time.

## Command: `/ci:landscape`

Full market map across every tracked competitor. Use for onboarding a new stakeholder, a board/leadership update, or the quarterly reassessment.

1. Read the stored competitor list (tier-1 and tier-2) and each competitor's most recent stored brief.
2. For each competitor, pull the current state of all 8 dimensions, citing sources per the Operating Rules.
3. Group output by market segment or competitive angle (e.g. "direct feature-parity competitors," "price-anchored challengers," "adjacent-category encroachers") rather than a flat alphabetical list — the landscape is a map, not a directory.
4. Flag any competitor whose classification (tier-1/tier-2) looks stale given recent wins/losses data, but don't silently reclassify — surface it as a recommendation.
5. Close with a "what changed since the last landscape run" summary if a prior landscape output exists, same diff discipline as single-competitor updates.

## Command: `/ci:battlecard [competitor]`

Generates or updates the sales-facing battlecard for one named competitor.

1. Pull the competitor's current stored profile (or run a full scan first if none exists — a battlecard needs the underlying profile to compress).
2. Compress into this structure:
   - **One-line positioning contrast** (how we differ, in a sentence a rep can say verbatim)
   - **Where we win** (sourced from actual win reasons, not assumption)
   - **Where we lose / their strength** (sourced from actual loss reasons — this is the uncomfortable but necessary section, don't soften it)
   - **How they'll pitch against us** (inferred from their messaging dimension, cited)
   - **Objection handling** — 2-3 specific rebuttal lines tied to real, sourced facts (a pricing structure difference, a feature gap, a review-site complaint pattern), not generic sales talk
   - **Landmines** — questions or framings that predictably lose the deal if a rep doesn't know them going in
3. Every line traces back to a source in the underlying profile. If a section has no sourced evidence yet, mark it `needs data` rather than inventing a plausible-sounding rebuttal.
4. On a repeat run, only touch sections whose underlying dimension changed (per the diff), and flag what changed at the top of the card so a rep who already memorized the old version knows what's different.

## Command: `/ci:winloss`

Pulls competitive reasoning specifically from won/lost deal data.

1. Read whatever deal data is available (pasted CRM export, closed-lost/closed-won records with competitor-mentioned field, or deal notes).
2. Group by competitor. For each: count of wins, count of losses, and the stated or inferable reason for each outcome.
3. Look for a pattern, not just a tally — is a competitor consistently winning on price, on a specific feature, on incumbency, on a segment we're weak in? Name the pattern explicitly, don't just list the raw counts.
4. Feed pattern findings back into the relevant dimension (a consistent price-loss pattern updates the Pricing dimension's severity; a consistent feature-loss pattern updates Product) so the next battlecard or landscape run reflects it.
5. If deal data doesn't name a competitor or doesn't state a reason, don't backfill a guess — count it as an unattributed loss/win and say so. An honest gap is more useful than a fabricated pattern.

## Command: `/ci:update [competitor]`

Single-competitor refresh, the core of the diff mechanic.

1. **Check for a stored prior brief for this competitor.**
   - **None exists**: this is a first run. Do a full 8-dimension scan, cite sources per dimension (minimum 3 source types), store the resulting brief as the new baseline. Output the full profile.
   - **A prior brief exists**: this is a diff run. Pull fresh source material for all 8 dimensions, compare against the stored brief dimension by dimension, and output **only what changed**, capped at roughly 300 words. Store the new state as the updated baseline.
2. For each changed dimension, assign a severity:
   - **RED** — costing deals now or needs a response this week (pricing cut, a feature-parity gap that just showed up in a live loss, a funding round that materially changes their sales capacity)
   - **YELLOW** — worth tracking, no immediate action (an adjacent hire, a minor messaging shift, a small feature addition)
   - **GREEN** — noted, no action implied (a blog post, a UI refresh, a hire in an unrelated function)
3. If nothing changed on a dimension, don't mention it. The output is the delta, not a checklist of "no change" lines across all 8 dimensions — that defeats the point of a ~300-word cap.
4. If the update was triggered by an event (funding, pricing, launch) rather than the monthly cadence, label it as an event-triggered run and note the trigger explicitly at the top.

## Command: `/ci:map`

Visual/tabular landscape overview, the compressed sibling of `/ci:landscape`.

1. Pull the current stored state (not a fresh full scan) for every tracked competitor.
2. Render as a table: competitor, tier, last-updated date, current severity flags by dimension (a quick-scan row of R/Y/G per dimension), and a one-line "why they matter right now."
3. Sort tier-1 competitors first, then tier-2, then flag any competitor whose brief is stale relative to its cadence (a tier-1 competitor with no update in the last 5+ weeks is overdue for the monthly review).
4. This command never triggers a new scan — it's a state readout of what's already stored. If a competitor has no stored data, the row says `no data yet` rather than being silently omitted.

## Worked Example (Rivergate Data)

**Background:** Marcus Iyer (Head of Growth Marketing) flagged this skill specifically after a competitor, **Flowbase**, quietly cut list pricing 15% last quarter. Rivergate found out when a mid-funnel deal cited Flowbase's new price as the reason for the loss — not from any monitoring. Marcus's standing ask: pricing gets watched continuously, not just reviewed once a month.

**`/ci:update Flowbase` — first run (no prior brief):**

Full scan across all 8 dimensions, minimum 3 source types cited:
- **Pricing**: $89/seat/mo listed on flowbase.io/pricing, screenshot dated 2026-06-30. Source: website.
- **Product**: added a native Slack integration per their changelog, dated 2026-06-15. Source: website (changelog page).
- **Hiring**: 3 open "Enterprise AE" postings on their careers page as of 2026-06-30, up from 0 last quarter — signals an upmarket push. Source: job postings.
- **Wins (against us)**: 1 known loss (Q2, mid-market deal, price-cited) per CRM closed-lost note. Source: deal record.
- Remaining dimensions (funding, partnerships, messaging, losses) noted with available sourced facts or marked `no data yet` where nothing was found.

Baseline stored. Output is the full profile (this is the exception to the 300-word cap — first runs are full scans by design).

**Event-triggered run, one week later — Flowbase drops pricing again:**

Trigger: pricing-page monitoring (intended integration) would have caught this; in this demo, Marcus pasted a screenshot showing $75/seat/mo, dated 2026-07-08.

```
Flowbase — event-triggered update (pricing change) — 2026-07-08

RED — Pricing: list price dropped $89 → $75/seat/mo (16% cut), source:
flowbase.io/pricing screenshot dated 2026-07-08. Second cut in two
quarters — pattern is aggressive price-led land-grab, not a one-off
promo. Recommend: flag to Marcus for this month's channel review,
update the battlecard's pricing objection-handling line before the
next live deal that mentions Flowbase.

No other dimensions changed since the 2026-06-30 baseline.
```

That's the full diff output — roughly 60 words, not a re-dump of all 8 dimensions, and it's the exact miss (a quiet pricing cut surfacing only in a lost deal) that Marcus asked this skill to close.

**`/ci:battlecard Flowbase` — updated after the pricing move:**

The card's "Where we lose / their strength" section and "Objection handling" section get touched (price is now a sharper wedge for them); everything else on the card stays as-is. The top of the card notes: *"Updated 2026-07-08 — Flowbase cut pricing to $75/seat, second cut in two quarters. Rep talk track: reframe on total cost of ownership and migration risk, not sticker price alone."*

## Rules

- First scan of a competitor is full and gets stored as the baseline. Every run after that is diff-only, capped at roughly 300 words, and reports only what changed.
- Every claim cites a specific source (website copy, a review quote, a job posting, a pricing-page screenshot with a date). No source, no claim — mark it unverified instead.
- Minimum 3 source types per competitor profile: website, reviews, job postings, at minimum.
- Pricing, funding rounds, and major launches get the 48-hour event-triggered window regardless of the monthly/quarterly calendar — this is the direct fix for the Flowbase-style miss.
- Severity (red/yellow/green) reflects actual deal impact and urgency, not how interesting or recent the change is.
- Win/loss reasoning comes from actual deal data. An unattributed loss stays unattributed — don't manufacture a competitor-driven narrative to fill a gap.
- A battlecard is a compressed sales tool. If a section has no sourced evidence, it says `needs data`, it doesn't get filled with generic sales language.
- `/ci:map` never triggers a new scan — it's a readout of stored state, and it says so plainly when a competitor has no data yet.
