---
name: collateral-generator
description: Generate personalized sales collateral (one-pagers, proposal decks, case-study snippets) at scale from a structured account table, one document per row from a shared template, then run a mandatory validation pass before anything is called done. Structural checks for docs (no unfilled template variables, no leaked placeholder text), visual re-render checks for anything with layout (deck/one-pager), repeated until zero new issues. Never fabricates a metric, gaps get marked explicitly. Run after signal-prospecting/waterfall-enrichment has produced a qualified-account table, or whenever the ask is "build me N personalized [asset]" instead of one at a time.
user_invocable: true
---

# Collateral Generator

Turns a table of qualified accounts into a folder of personalized, ready-to-send documents, without hand-writing each one and without letting quality drift as volume goes up. The two failure modes this exists to prevent: (1) collateral that's the same doc with a find-replaced logo, and (2) collateral that looks finished but has a broken template variable, a leaked `{{placeholder}}`, or a made-up number sitting in the third paragraph.

**The genuine gap this fills:** every generate-a-document skill I found in public circulation handles one document at a time, and the batch patterns I've built for outreach elsewhere handle rows of short text, not full documents. I didn't find a public pattern that wires row-by-row structured input straight through a generate-then-validate document pipeline end to end, so this skill is my own design rather than an adaptation, and it says so here instead of inventing a source. That combination, not either half alone, is the actual "thought about scale" move.

## Operating Rules (read first)

- **One row, one document, one shared template.** Never write account-specific documents freeform. If the template doesn't have a slot for something the account needs, that's a template gap, fix the template, don't patch around it in one file.
- **Generate via a scripting library, not a raw prose dump.** Docs and decks get built by a script that populates a template's structured fields (a docx/pptx generation library, or Slides API calls), not by writing paragraphs directly into a finished-looking document and hoping the formatting holds. This is what makes the validation pass in the next rule possible at all. You can't structurally validate free-hand prose.
- **Validation is mandatory and it is not the same step as generation.** Generation produces a candidate. Validation is a separate pass that either passes or fails it. A document that hasn't been through validation is not done, regardless of how complete it looks.
- **Never fabricate a metric, a quote, or a case-study result.** If a data point isn't present in the source row, the field is rendered as `[METRIC NEEDED - not in source data]`, not filled with something plausible. A gap flagged is a passing document. A gap invented is a failed one, even if nobody would notice.
- **The narrative shape for case-study-style collateral is fixed: Overview → Challenge → Process → Solution → Impact → Reflection.** Six parts, in that order, every time. Don't let a template author collapse it to three sections because a given account's story is thin, thin sections are fine, missing sections aren't.
- **Fix root causes in the template, not in individual outputs.** If the same field is empty across five accounts, that's a source-data gap or a template bug, not five documents to patch by hand.
- **This skill produces drafts for a human send decision, not sends.** Nothing generated here goes to a prospect without the same approval gate the rest of this repo uses for anything external-facing.

## Intended Integrations (not live in this demo)

- **A docx-generation library** (e.g. `python-docx` or a template-merge library) would drive one-pager and case-study generation, populating a fixed template's fields programmatically rather than writing prose into a finished file. This mirrors Anthropic's own first-party docx/pptx skills (`anthropics/skills`), which use exactly this generate-via-library approach specifically so the output stays structurally checkable.
- **Google Slides API** would drive proposal-deck generation for the same reason: a script filling named placeholders in a deck template, not a model drawing slides freehand.
- **Qwilr or PandaDoc** would be the delivery/tracking layer once a document is validated, turning the local file into a trackable link with view analytics, sitting downstream of everything this skill does.
- None of this is wired into the demo repo. Right now, "generation" means building the account's field data as structured JSON per row and rendering it into Markdown that mirrors the target document's real section structure; "validation" runs against that Markdown and its field data, using the exact same rules a real docx/pptx validation pass would use. The rules and the gate are real. The rendering backend is stated, not implemented.

## Step 1: Read the Account Table

Read the qualified-account table (CSV, CRM export, or Markdown table), one row per account. This is intended to be the output of `signal-prospecting` → `waterfall-enrichment`: each row should already carry company name, contact name/title, industry, the specific trigger/signal that qualified them, and whatever enrichment data (funding stage, headcount, tech stack, recent news) survived the waterfall.

Before generating anything, check the table for the fields the chosen template actually needs. Missing fields are not blockers, they become `[FIELD NEEDED - not in source data]` markers at generation time, but note them now so the run summary isn't the first place anyone sees the gap.

## Step 2: Pick the Template and Asset Type

Three supported asset types, one shared structure each:

- **One-pager**: single-page account brief, used as a pre-call leave-behind. Sections: Overview, the specific trigger that qualified them, proposed angle, next step.
- **Proposal deck**: multi-slide, used post-discovery-call. Sections: Overview, Challenge, Process, Solution, Impact (projected, clearly labeled as projected if not yet delivered), Reflection/next-step slide.
- **Case-study snippet**: short narrative block for embedding in a deck or one-pager, used when there's a comparable existing client story to adapt. Always the full six-part shape: **Overview → Challenge → Process → Solution → Impact → Reflection**. This shape doesn't flex per account. If an account's story is thin on one part (e.g. no measured Impact yet), that section stays present and explicitly marked as a gap, it doesn't get quietly dropped to make the document look more complete than the data supports.

Confirm the asset type once at the start of a batch run, don't ask per row.

## Step 3: Generate Per Row (candidate pass)

For each account row, render the template with that row's fields:

1. Populate every template variable from the row's structured data.
2. Any variable with no source value becomes an explicit gap marker (`[FIELD NEEDED - not in source data]` or `[METRIC NEEDED - not in source data]`), never a plausible-sounding fill.
3. Write the candidate to `.tmp/collateral/<batch-id>/<account-slug>.md` (standing in for the real docx/pptx output in this demo). Naming: lowercased company name, spaces to hyphens, e.g. `rivergate-data.md`.
4. Do not mark a candidate as final yet. Generation only produces something eligible for validation.

This is a mechanical loop, not a fresh creative pass per account. The personalization comes from the row's real data slotted into fixed sections, not from writing each document from scratch.

## Step 4: Validate (mandatory gate, not optional polish)

Run two validation passes on every candidate, and don't skip either because a document "looks right":

**Structural validation (every asset type):**
- No unresolved template variable syntax remains (no stray `{{...}}`, no unrendered field names).
- No gap marker was silently deleted or overwritten with invented content between generation and this check.
- All six narrative sections are present for case-study-style content, even if a section is short.
- No dollar figure, percentage, or named metric appears anywhere in the document that isn't traceable to a field in the source row.

**Visual/layout validation (deck and one-pager only, anything with real layout):**
- Re-render the candidate and inspect it for overflow, truncated text, broken section breaks, or a slide/page that doesn't match the template's intended structure.
- Fix the issue in the template or generation script, not by hand-patching the rendered output.
- Re-render again. Repeat until a pass finds zero new issues. A single clean pass isn't enough if the first pass found problems, the fix itself needs re-checking, not just the field it touched.

A candidate that fails either pass does not get promoted. It's logged as a validation failure in the run summary (Step 5) with the specific reason, and the batch moves on to the next account rather than stalling on one row.

## Step 5: Write the Run Summary

One file per batch, alongside the per-account outputs: `.tmp/collateral/<batch-id>/run-summary.md`.

```
Collateral run - <batch-id> - <asset type> - <date>

SUCCEEDED (no gaps, validation clean)
- <account>: <output filename>

SUCCEEDED WITH GAPS FLAGGED
- <account>: <output filename>, missing: <field list>

FAILED VALIDATION
- <account>: <reason>, not written to final output

Totals: N succeeded, N with gaps, N failed, N total rows
```

This is the single place someone checks before a batch goes anywhere near a send queue. "Succeeded with gaps flagged" is not a failure state, it's an honest one, but it should never be silently treated as identical to a clean pass.

## Worked Example (Rivergate Data, fictional)

Batch of 3 accounts from a `waterfall-enrichment` output, asset type: one-pager.

Row 1, Rivergate Data (Series A B2B SaaS, signal: just closed Series A, hiring first AE): all required fields present. Candidate generated, structural validation clean, no layout issues on re-render. Promoted.

Row 2, a second account: enrichment table has company name, contact, industry, but the "specific trigger" field is blank, the waterfall didn't find a qualifying signal beyond firmographic fit. Candidate generated with `[FIELD NEEDED - not in source data]` in the trigger slot. Structural validation passes (the gap is marked, not invented), layout is clean. Promoted as "succeeded with gaps flagged."

Row 3, a third account: template variable for "proposed angle" failed to resolve due to a malformed row (a stray comma split the CSV field). Structural validation catches the unresolved `{{proposed_angle}}` syntax. Fails validation. Logged, not written to a final file. Root cause noted for the batch operator: fix the source row, not the rendered output.

```
Collateral run - 2026-07-02-batch1 - one-pager - 2026-07-02

SUCCEEDED (no gaps, validation clean)
- Rivergate Data: rivergate-data.md

SUCCEEDED WITH GAPS FLAGGED
- [Account 2]: [account-2].md, missing: qualifying_trigger

FAILED VALIDATION
- [Account 3]: unresolved template variable {{proposed_angle}}, malformed source row, not written to final output

Totals: 1 succeeded, 1 with gaps, 1 failed, 3 total rows
```

## Rules

- One shared template per asset type. Personalization comes from row data, not from rewriting the template per account.
- Generate via structured field population, never raw prose dumped straight into a finished-looking document.
- Validation is a separate, mandatory step. A candidate that hasn't passed it is not done, no matter how complete it looks.
- Any metric, quote, or result not present in the source row is a gap marker, never an invented plausible number.
- The six-part case-study shape (Overview → Challenge → Process → Solution → Impact → Reflection) never flexes to skip a section, thin sections stay visible as gaps instead.
- Visual/layout re-checks repeat until a pass finds zero new issues, not just once.
- A failed-validation row gets logged and skipped, it doesn't block or corrupt the rest of the batch.
- Nothing generated here is sent externally without the same explicit approval gate as everything else in this repo.
