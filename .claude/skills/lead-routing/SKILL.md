---
name: lead-routing
description: Score every new or updated lead on two dimensions, fit (firmographic match to ICP) and engagement (behavioral signal strength), advance it through the Subscriber-to-Customer lifecycle when it crosses stage thresholds, then route it to an owner using the right method for that lead type (round-robin, territory, ABM, or skill-based). Inbound-hot leads carry a 5-minute contact SLA with a defined escalation path. Run whenever a lead is created, updates, or crosses a scoring threshold, or when asked to "score this lead," "route this lead," "why didn't this get assigned," or "check SLA breaches."
user_invocable: true
---

# Lead Scoring & Routing

Adapted from coreyhaines31's `marketingskills` repo (github.com/coreyhaines31/marketingskills), `skills/revops/SKILL.md`: the lifecycle state machine, dual-dimension scoring model, 4-method routing decision tree, and SLA escalation logic are that skill's design. This version ports the mechanics into an operator-OS shape (explicit steps, worked example, stated-not-wired integrations) for the Rivergate Data demo.

A lead scored wrong either burns a rep's time on a bad fit or lets a hot buyer go cold waiting for a human. This skill exists so every lead gets the same evaluation, every time, instead of whichever rep glances at it first.

**Expansion leads route here too.** When `lifecycle-signal-tracker` flags an existing account as expansion-ready (fast time-to-value, usage above contracted seats), that becomes a routable lead in this skill. It skips the fit/engagement scoring built for net-new leads and always routes to the current account owner via the ABM/named-account rule in Step 5, never round-robin, because an expansion play on an existing logo belongs to whoever already owns the relationship.

## Operating Rules (read first)

- **Score fit and engagement separately, never blend them into one number until the routing step.** A high-fit lead with zero engagement is a nurture target, not a rep's problem yet. A low-fit lead with high engagement (a student, a competitor, a vendor) is still not a lead. Don't let engagement alone promote something the ICP already ruled out.
- **Stage transitions require the stated criteria, not a vibe.** A lead doesn't become an MQL because it "feels warm." It becomes an MQL because it crossed the fit + engagement thresholds defined in Step 2.
- **The SLA clock only starts on inbound-hot leads**, defined in Step 4. Don't apply the 5-minute standard to every lead. It would be ignored within a week and the real fires would stop getting attention.
- **Routing method is chosen by lead type, not by whichever method is fastest to reason about.** Check ABM account ownership before round-robin. A named-account lead skips the round-robin queue even if it's that rep's "turn."
- **Never assign and go silent.** Every routing decision gets a one-line reason attached (which method fired and why) so a rep or manager can audit it without re-deriving the score.
- **This skill doesn't fabricate CRM data.** If a required field (company size, engagement event, territory) is missing, say so and either score what's available or flag the lead as unscoreable. Don't guess a value to force a clean tier.

## Intended Integrations (not live in this demo)

- **HubSpot or Salesforce workflow API** would own the actual lifecycle-stage field and fire the stage-change automation (Step 2) the moment a lead crosses a threshold, instead of this skill computing it against a pasted export.
- **Clay or a similar enrichment tool** would populate the firmographic fields (employee count, industry, tech stack, funding stage) that feed the fit score, so scoring runs against fresh data instead of whatever was captured at form-fill.
- **Segment or a CDP** would stream the behavioral events (page views, email opens/clicks, content downloads, pricing-page visits) that feed the engagement score in near-real time, instead of a batch pull.
- **Slack (via webhook or app)** would deliver the assignment ping to the rep and the escalation alert to the manager (Step 4), with a deep link straight to the lead record.
- None of these are wired into this repo. This is a demonstration of the scoring and routing logic. Right now this skill runs on whatever lead data is pasted or exported, and states plainly when a step would depend on a live integration it doesn't have.

## Step 1: Pull the Lead Record

Take the lead as given, pasted fields, a CSV row, or a CRM export. Note what's present and what's missing. Required for scoring: company size, industry, current lifecycle stage, and whatever behavioral events are on file (visits, opens, clicks, downloads, demo requests). If firmographic fields are missing, flag that fit scoring will be partial. If no behavioral events are on file, engagement scores as zero, not unknown, a lead with no recorded activity has not engaged.

## Step 2: Score Fit (Explicit, ICP-Match)

Fit is what the lead *is*, not what they've done. Score against Rivergate Data's ICP:

| Signal | Points | Rivergate ICP criteria |
|---|---|---|
| Company size | 0–25 | 50–500 employees = 25; 20–49 or 501–2000 = 12; outside range = 0 |
| Industry | 0–20 | B2B SaaS / data infra = 20; adjacent (fintech, martech) = 10; unrelated = 0 |
| Tech stack signal | 0–15 | Already runs a competing or adjacent tool = 15; no signal = 0 |
| Role/seniority | 0–20 | VP+ or Director in Sales/RevOps/Data = 20; manager-level = 10; IC or unrelated function = 0 |
| Funding stage | 0–10 | Series A–C = 10; pre-seed/seed or Series D+ = 5; public/bootstrapped = 0 |
| Geography | 0–10 | Core territory (US/UK) = 10; secondary = 5; unsupported region = 0 |

**Fit score = sum, 0–100.** Fit tiers: **High (70+)**, **Medium (40–69)**, **Low (<40)**.

## Step 3: Score Engagement (Implicit, Behavioral)

Engagement is what the lead has *done*. Score recent activity (rolling 30 days unless the lead record states otherwise):

| Signal | Points |
|---|---|
| Pricing page visit | +15 |
| Demo/contact form request | +25 |
| Content download (whitepaper, template) | +8 (max 16 across multiple) |
| Email open | +2 (max 10 across multiple) |
| Email click-through | +6 (max 18 across multiple) |
| Webinar attendance | +12 |
| 3+ site visits in 7 days | +10 |
| No activity in 30+ days | Cap total at existing score, no growth, and flag for decay review |

**Engagement score = sum, 0–100 (cap at 100).** Engagement tiers: **Hot (60+)**, **Warm (25–59)**, **Cold (<25)**.

## Step 4: Advance the Lifecycle Stage

Stage transitions require crossing both dimensions where stated. Never skip a stage; a lead can move fast through one, but it still passes through it.

| Stage | Entry criteria |
|---|---|
| **Subscriber** | Opted into any list (newsletter, resource download) with no fit data yet, or fit data present but only Low tier and no sales-relevant action taken. |
| **Lead** | Fit score computed (any tier) and at least one identifiable engagement event on record. |
| **MQL** (Marketing Qualified) | Fit = Medium or High **and** Engagement = Warm or Hot. This is the threshold that hands the lead to sales for the first time. |
| **SQL** (Sales Qualified) | MQL criteria met **and** a rep or SDR has confirmed BANT-equivalent fit in a live conversation (call, chat, qualifying email reply), not just score-based. This step requires a human confirmation event on the record; scoring alone cannot promote past MQL. |
| **Opportunity** | SQL confirmed **and** a scoped deal exists in the CRM pipeline (defined next step, stage, or amount). |
| **Customer** | Opportunity closed-won. |

If a lead's stage in the source record disagrees with what the criteria above say it should be, flag the mismatch rather than silently overwriting it. That's a data-hygiene gap for the rep or ops owner to resolve, same principle as cross-file disagreement in any other skill in this repo.

**Inbound-hot definition:** a lead that hits Fit = High and Engagement = Hot *from a single inbound action* is inbound-hot regardless of current lifecycle stage. This is the trigger for the SLA in Step 6.

**Named override (demo request + pricing visit, same session):** a same-session demo request combined with a pricing-page visit also qualifies as inbound-hot at Fit = High, even when the raw engagement point sum lands in Warm (25-59) rather than clearing the 60-point Hot threshold on its own. A demo request is a stronger buying signal than the additive point scale gives it credit for. Someone who visited pricing and asked for a demo in the same session isn't a "40-point lead," they're a lead who just told you they're ready, and the scoring table wasn't built to weight that combination correctly on its own. This is the same philosophy as the named job-change-plus-funding stack in `signal-prospecting`: a specific, evidence-backed combo gets named special handling instead of being left to the raw additive score. The difference is that here the handling actually changes the outcome (40 points would otherwise read Warm, not inbound-hot), where in `signal-prospecting` the combo already clears its threshold on points and the naming is about queue priority. Either way it's a named exception, not a silent one. No other Warm-tier engagement combination qualifies as inbound-hot without a same-session demo request specifically. A lead sitting at 40 points from webinar attendance and email clicks, with no demo request, is Warm, not inbound-hot.

## Step 5: Choose the Routing Method

Check in this order. The first method that applies wins. Don't average methods or let a later one override an earlier match.

1. **ABM / named-account.** Is the lead's company on Rivergate's named-account list, or does an existing owner already hold other contacts at that company? If yes, route to that owner regardless of round-robin position, territory, or skill fit. Named-account ownership overrides everything else. A rep working a $200k target account should get every lead from that logo, not just the ones that land on their round-robin turn.
2. **Territory-based.** If no named-account owner exists, does the lead's geography or segment map to a rep with defined territory ownership (e.g., a UK-specific AE, an enterprise-segment AE)? If yes, route there.
3. **Skill-based.** If no territory owner applies, does the lead's stated need require product-line expertise a specific rep or team owns (e.g., a data-infra-specific technical evaluation vs. a general SaaS buyer)? If yes, route to the rep with that expertise.
4. **Round-robin (default).** If none of the above apply, route to the next rep in the fair-distribution queue. This is the fallback, not the first check, and it exists so leads never sit unassigned while the other three checks are being reasoned about.

Record which method fired and why in one line attached to the assignment (see worked example).

## Step 6: SLA and Escalation (Inbound-Hot Only)

- Any lead marked **inbound-hot** (Step 4) starts a **5-minute contact-window SLA** the moment it's routed. This threshold exists because Dana Whitfield (VP Sales) approved it on 2026-06-23 after response-time data showed conversion drops sharply past 5 minutes. See the worked example below for how that decision shaped this skill.
- If the assigned rep has not logged a contact attempt (call, email, or CRM touch) within 5 minutes, escalate:
  1. **First escalation (5–10 min):** reassign the lead to the next available rep in the same routing pool (same territory/skill/ABM owner if a backup exists, otherwise round-robin) and alert the original rep that the lead moved.
  2. **Second escalation (10+ min, or no backup rep available):** alert the sales manager directly (Dana, or whoever owns that rep) with the lead's fit/engagement scores and the elapsed time, so a manager can intervene manually rather than the lead sitting further.
- Non-inbound-hot leads don't carry this SLA. Applying a 5-minute clock to every MQL would make the alert meaningless within a week. The whole point of the tier is that it's rare and it means something when it fires.

## Worked Example (Rivergate Data, fictional)

A lead comes in from the pricing page: **Jordan Reyes, VP RevOps at a 220-person Series B B2B SaaS company (US), requested a demo within the same session as visiting pricing.**

**Step 2, Fit:**
Company size 220 → 25. Industry B2B SaaS → 20. Tech stack: currently runs a competing tool → 15. Role: VP RevOps → 20. Funding: Series B → 10. Geography: US → 10.
**Fit = 100 → High.**

**Step 3, Engagement:**
Pricing page visit → +15. Demo request → +25. Same session, no other history.
**Engagement = 40 (Warm).** Under the 60-point Hot threshold on the raw scale, but the demo-request-plus-pricing-visit combination triggers the Named override in Step 4, not the base Fit-High-and-Engagement-Hot rule.

**Step 4, Lifecycle:**
No prior record for Jordan. Fit High + Engagement present → enters as **Lead**, then immediately **MQL** (Fit High + Engagement Warm at 40 points clears the MQL threshold on its own). Flagged **inbound-hot** per the Named override (Fit High + same-session demo request + pricing visit: the override is what lifts this past Warm into inbound-hot, not the MQL call).

**Step 5, Routing:**
No named-account match (new company, no existing Rivergate contact). No territory owner specifically assigned to this segment. Skill check: nothing product-specific stated in the demo request. Falls to round-robin. Next rep in queue: **Priya Nandakumar**.
Assignment reason logged: *"Round-robin (no ABM/territory/skill match) → Priya Nandakumar. Inbound-hot: Fit 100 (High), pricing+demo combo."*

**Step 6, SLA:**
5-minute contact clock starts at assignment. At 6 minutes, no contact logged. **First escalation fires**: lead reassigned to the next available rep in the round-robin pool, Priya alerted that it moved. This is the exact mechanic Dana approved on 2026-06-23. The routing logic didn't change, only the enforcement of how fast the first touch has to happen once it lands.

## Rules

- Fit and engagement are scored independently. Never let a high score on one dimension substitute for the other when deciding lifecycle stage.
- SQL requires a human-confirmed qualifying conversation on record. Scoring alone cannot promote a lead past MQL.
- ABM ownership beats round-robin every time, even mid-queue. Check it first, not last.
- The 5-minute SLA applies only to inbound-hot leads. Don't let scope creep turn every MQL into a "fire."
- Every assignment carries a one-line reason: which method fired, and the fit/engagement scores that drove it. No silent routing.
- If a required field is missing, say so and score what's available, or flag the lead unscoreable. Never invent a firmographic or behavioral value to force a clean tier.
- If the source CRM's stated stage disagrees with what the criteria in Step 4 produce, flag the mismatch rather than silently overwriting either value.
