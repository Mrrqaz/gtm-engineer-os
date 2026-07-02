# Role Target (swappable per application)

> This is the one file I customize per company I apply to. Everything else in the repo stays constant. Currently filled with placeholder/example structure — fill in real company details before sending.

## Who I'd support

- **Company:** [target company]
- **Stage:** [pre-seed / seed / Series A / etc.]
- **Team I'd sit in:** [GTM / RevOps / Growth]
- **Who I'd report to:** [name/role, or "unknown — ask on the call"]

## JD summary (what they actually asked for)

[Paste the 4-6 concrete recurring tasks from the JD here, verbs not adjectives — e.g. "builds Clay tables," "owns HubSpot data hygiene," "writes n8n workflows"]

## First 90 days (draft)

- **Days 1-30:** audit the current stack (CRM state, enrichment sources, sequence performance) using `crm-hygiene-audit` and `pipeline-forecast-review` as the starting lens. Find where pipeline is actually leaking before proposing anything.
- **Days 31-60:** ship one small, real system fix from the audit — likely a dedup/routing cleanup or a signal-based prospecting motion — and get it running end to end with a human approval gate on sends.
- **Days 61-90:** propose the next system to build based on what the 90-day audit surfaced, backed by the `growth-experiment-prioritizer` ICE/RICE framing rather than a gut call.

## Why I'm a fit (honest, including gaps)

- I've built and run the real version of most of what's in this repo through Autoage: enrichment waterfalls, signal-based outbound, CRM data hygiene, reply classification, campaign performance tracking.
- Gap to name upfront: [e.g. "no formal Salesforce admin certification" / "SQL is functional, not expert" — fill in honestly per company, don't oversell]
- I'm not pitching "I've done this exact stack before." I'm pitching that I build these systems the same way regardless of which specific tools are involved, and I pick up new tools fast because I understand the underlying mechanics (waterfall enrichment, scoring, routing logic), not just button locations in one vendor's UI.

## Interview questions I'd ask them

- What's the biggest pipeline leak right now — top of funnel, mid-funnel routing/qualification, or late-stage forecast accuracy?
- What does the current stack actually look like end to end, and where does a human still manually bridge two tools?
- Is this role expected to write code (Python/SQL/API) day one, or grow into it?
