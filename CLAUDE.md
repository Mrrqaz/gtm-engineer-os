# GTM Engineer: Operator Identity

This is a working Claude Code operating system built for a GTM Engineer / Growth Engineer / RevOps Engineer role. It exists to demonstrate how I'd actually run the job on day one, not to describe it in prose.

## Job description (bounded)

A GTM Engineer builds and maintains the automated systems that generate and move pipeline: enrichment waterfalls, signal-based prospecting, CRM data integrity, lead routing, outbound sequence performance, pipeline forecasting, competitive intelligence, growth experimentation, lifecycle/retention signal tracking, and sales collateral at scale. The job is closer to "part AE, part sales engineer, part data engineer" than a marketing role. The output is pipeline, not campaigns.

## Operating principles

1. **Systems over single tasks.** Every skill here is a repeatable workflow, not a one-off prompt. If I do something twice, it becomes a skill.
2. **Evidence-gated, not vibes-gated.** Every skill below has explicit thresholds, scoring formulas, or decision rules, not "use your judgment" prompts. Judgment calls are named, not hidden.
3. **Spend-aware.** Enrichment and API calls cost money. Skills gate on fit *before* spending (see `waterfall-enrichment`), not after.
4. **Autonomous inside the job, gated at the edge.** I can run scoring, drafting, and analysis freely. Anything that touches a live send, a CRM write, or spend crosses an approval gate.
5. **Integrations are stated, not wired.** Every skill names the real tool/API/MCP it would connect to (Clay, HubSpot/Salesforce, Apollo, n8n, a data warehouse). None of it is live here. This repo is a demonstration of how I'd build the system, not a working production pipeline. Wiring happens once I'm actually doing the job.
6. **Diff-only where recurrence matters.** Monitoring skills (competitive intel, lifecycle signals) compare against their own prior run and report only what changed: a real monitor, not a report generator that re-dumps everything each time.

## File map

- `context/goals.md`: my real goals for this job search (not fictional)
- `context/role-target.md`: swappable per-application file (who I'd support, first-90-days plan, honest fit assessment)
- `context/people/`: fictional stakeholder profiles for the worked-example company, "Rivergate Data"
- `context/decisions-log.md`: fictional seeded decision log showing how the skills below get used in practice
- `.claude/skills/`: the 12 skills themselves

## Worked example company

All worked examples in the skills below reference a fictional Series A B2B SaaS company, **Rivergate Data**, and three fictional stakeholders (`context/people/`). The scenario is fictional; my goals, principles, and the skill design are real.
