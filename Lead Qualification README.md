# Lead Qualification & CRM Sync Engine

A Make.com automation that takes inbound leads from a web form, verifies they are not duplicate submissions, enriches them with real company data, scores them against an explicit qualification rubric using Gemini, and routes high-priority leads to Slack in real time.

**A note on the tool stack.** Every tool used in this build is currently on its free tier. This reflects the scale it was built and tested at, not a ceiling on what the architecture can support. The design itself does not change if a client needs more volume than a free tier allows; only the specific tool at each stage would be swapped for a paid tier of the same tool, or a comparable paid alternative. Where that swap would happen, and why, is called out in the Tools section below.

## The problem this solves

Manually reviewing inbound leads does not scale. A sales rep or founder ends up giving equal attention to a curious visitor and a company that is ready to buy, simply because both arrived in the same inbox in submission order. This system automates the triage step: it filters out duplicate submissions before spending any API budget on them, researches who the lead actually represents, and scores that lead against a defined set of criteria before a human ever looks at it. The leads most worth a human's time surface first, instead of getting buried in the noise of routine inquiries.

This is scoped for small-to-mid inbound volume, roughly tens to a few hundred leads per week on the free tiers used here. It is not a replacement for enterprise lead routing systems with SLA enforcement or multi-channel intake. See Scope and Limitations below.

## How it works

**Intake.** A Tally form collects four fields from an inbound lead: name, email, company website, and message. Submission triggers a Make webhook.

**Duplicate check.** Before any paid or rate-limited API call runs, the system searches Airtable for an existing record with the same email. If one exists, the pipeline stops immediately. No enrichment call, no scoring call, no new record. This ordering was a deliberate correction made mid-build: an earlier version of this pipeline created a record and ran enrichment and scoring on every submission before checking for duplicates, which would have wasted enrichment credits and AI calls on repeat submissions and left redundant records behind for every duplicate. Gating on duplicate status before any paid API call is the kind of sequencing decision that determines whether a system can run at real volume without burning through its API budget on redundant work.

**Enrichment.** For genuinely new leads, the system calls the Hunter.io Domain Search API with the company domain the lead provided. This returns the actual registered company name behind that domain, which is often different from what the domain implies (`squareup.com` resolves to the company Square, for example). This step turns a bare URL into an identifiable company before scoring happens.

**Scoring.** A Make tool module calculates three deterministic signals with no AI involved: whether enrichment found a real company, whether the message contains explicit buying-intent language (pricing, demo, buy, quote, purchase), and the character length of the message. These three signals, plus the lead's raw data, are passed to Gemini along with an explicit scoring rubric that defines exactly what combination of signals produces a Hot, Warm, or Cold classification, and instructs the model to reason through each factor before committing to a score rather than defaulting to a safe middle answer. The API call enforces structured output using Gemini's `responseSchema` parameter, restricting the `score` field to exactly `Hot`, `Warm`, or `Cold` at the API level rather than relying on the model to follow formatting instructions in the prompt text. Temperature is pinned to 0.1, since this is a classification task where the same input should reliably produce the same output, not a creative task that benefits from varied responses.

**Routing.** Leads scored Hot trigger an immediate Slack alert containing the lead's name, enriched company name, and Gemini's written reasoning. Warm and Cold leads are not pushed anywhere; they sit in Airtable with their score and reasoning for review on a normal cadence. This avoids notification fatigue while still surfacing the leads that need fast attention.

**Failure handling.** The Hunter.io call and the Gemini call each have an independent error handler attached directly to that module. If either fails, the affected lead's Airtable record is updated with a specific status (`Enrichment Failed` or `Scoring Failed`) rather than being lost or left in an ambiguous state. Make does not allow an error-handler branch to merge back into a module already receiving input from the main success path, so each failure path resolves on its own rather than trying to rejoin the primary flow.

## Why these choices, and what else was considered

**Make.com over n8n or Zapier.** Make's visual error-handling model, attaching a dedicated handler directly to the module that can fail, maps cleanly onto the independent-fallback-per-external-call requirement this build needed. Zapier was ruled out because its native code execution is limited on lower tiers, which becomes a real constraint the moment a workflow needs custom logic beyond simple field mapping. n8n was the closer alternative, since it has a free native code node that Make does not, but nothing in this specific build required custom code, and Make's native app connectors for Airtable and Slack are more mature than n8n's for this exact combination.

**Tally over Typeform or Google Forms.** Typeform's free tier does not include webhooks at all; webhook support is gated behind a paid plan. Google Forms works, but its nested response format (answers buried several layers deep in the payload, keyed by field ID rather than field name) makes downstream mapping more fragile than it needs to be for a webhook-driven pipeline like this one. Tally is less recognizable as a brand than Typeform, which is a real trade-off worth naming, but it was the option that kept both a genuine webhook and a genuine free tier intact without the mapping fragility of Google Forms.

**Airtable over a full CRM like HubSpot or GoHighLevel.** Airtable was chosen as the system of record here because its API is straightforward and predictable to build against, while still being a real, commonly used lightweight CRM layer for small businesses and agencies. For a client already running Salesforce or HubSpot, this system's Airtable write step would be swapped for a native write to that platform instead. The scoring and enrichment logic upstream of that step would not need to change.

**Hunter.io over Clearbit, Apollo, or ZoomInfo.** Most competing enrichment tools advertise a free tier that is actually a time-limited trial that converts to paid, not a genuinely recurring free allotment. Hunter.io's free tier renews monthly with no expiry and no credit card required. Apollo has a stronger free tier in some respects, but its core strength is contact-level data tied to people, not company-level domain lookups, which is what this specific enrichment step needed.

**Gemini over OpenAI or Claude.** For this task, the deciding factor was cost at volume: `gemini-3.5-flash-lite` is priced well below comparable models from other providers for a bounded classification task like this one, and Google's free tier for Flash-Lite comfortably covers testing and low-volume production use without a paid API key. For a task this narrowly scoped, with an explicit rubric already doing most of the reasoning work, the specific model provider matters less than the discipline of the prompt and the enforcement of structured output, both of which are provider-agnostic decisions and would carry over cleanly if a client had a preferred or existing AI vendor relationship.

**`gemini-3.5-flash-lite` over a larger Gemini model.** Lead scoring here is a bounded classification task with a fully explicit rubric already doing the heavy lifting. It does not need the deeper reasoning capacity of a larger model, and the price difference compounds meaningfully at volume. If this system were extended to a more open-ended judgment call, for example ranking leads against a full ideal-customer-profile document rather than a fixed rubric, a larger model would be worth evaluating against the added cost.

**Structured output enforcement over prompt instruction.** The first version of the scoring prompt simply asked Gemini to "respond in this exact JSON format" as plain text. That works most of the time, but nothing prevents the model from occasionally adding commentary or slightly malforming the response, which would break the parsing step downstream and fail a lead's scoring for no reason related to the lead itself. Passing a formal `responseSchema` moves that guarantee from the prompt, where it is a request, to the API contract, where it is enforced. This is a small technical detail, but it is the difference between hoping the model behaves and making it structurally unable to do otherwise.

**Hybrid scoring instead of a bare AI-guessed number.** The rubric is not "score this lead 1 to 10." It defines exact conditions for Hot, Warm, and Cold, weights buying-intent language as the strongest single signal, explicitly treats a missing enrichment result as neutral rather than penalizing small or new companies simply because Hunter does not have data on them yet, and instructs the model not to default to Warm as a lazy middle answer. This produces a system whose scoring logic can be explained and defended in front of a client, rather than a black box that occasionally produces a plausible-sounding number with no visible reasoning behind it.

## Scope and limitations

- Message-length thresholds (20 and 100 characters) in the scoring rubric are reasonable starting assumptions, not calibrated against real conversion data. They should be tuned once a client has actual outcome data showing which message lengths correlate with real deals.
- Hunter.io's free tier is rate-limited to 50 domain lookups per month. A production deployment at meaningful volume would need Hunter's Starter plan, which starts at $34 to $49 per month for 2,000 credits, or a comparable paid enrichment tool.
- No SLA enforcement or escalation timer on Hot leads beyond the initial Slack alert.
- Single intake channel (one Tally form). A production system would likely need to accept leads from multiple sources.
- No CRM sync beyond Airtable. This is a real limitation for a business already running Salesforce, HubSpot, or a similar platform, and would need an additional integration step, not a rebuild of the core logic.

## Cost and ROI

**Hunter.io (Domain Search).** Free tier: 50 lookups per month at no cost, the tier used throughout this build. Beyond that, Hunter's Starter plan is $34 to $49 per month for 2,000 credits, roughly $0.02 to $0.025 per lookup at that tier.

**Gemini (scoring call).** Using `gemini-3.5-flash-lite` at $0.30 per million input tokens and $2.50 per million output tokens. A real scoring call captured during this build consumed 526 input tokens and 89 output tokens, costing approximately $0.0004 per lead scored. At 500 leads per month, that is roughly $0.20 in AI cost for the entire scoring stage. Gemini also offers a free tier for Flash and Flash-Lite models with reduced rate limits, which is what this build ran on.

**Total marginal cost per lead**, once past both free tiers, is well under $0.03, dominated almost entirely by the enrichment lookup rather than the AI call.

**Manual labor comparison.** A person spending even two minutes per lead manually researching the company and deciding priority, at a conservative $15 per hour, costs $0.50 per lead in labor alone, before accounting for the inconsistency of manual judgment across a busy day or across different team members. At 500 leads per month, that is $250 in labor for a task this system performs for well under $15 once past the free tiers, with identical criteria applied to every lead regardless of who is reviewing it or what time of day it arrives.

## Enterprise scaling notes

Every tool in this build currently runs on a free tier, reflecting the scale it was tested at rather than a limit on the design. The scaling path for each piece is direct rather than a rebuild:

- Hunter.io free tier to Hunter Starter or Growth, or a comparable enrichment provider, once monthly lead volume exceeds 50.
- Gemini's free tier to its paid tier, which mainly raises rate limits; the per-call cost calculated above already reflects paid-tier pricing and stays low even at high volume.
- Airtable to a full CRM (Salesforce, HubSpot) if the client already runs one, replacing only the final write step.
- Tally to a client's existing intake form or landing page provider, since the webhook pattern this build uses works the same regardless of which form tool sends the payload.

Beyond swapping individual tools, a business operating at real scale would also need multi-tenant data isolation if this were serving multiple clients from one instance, throughput beyond Airtable's API rate limits, a proper audit trail on every scoring decision for compliance or dispute review, and role-based access control on who can view raw lead data versus aggregate scoring trends. None of these are difficult to add on top of the current architecture, but they are not built in today, since they depend on the specific compliance and access requirements of the client deploying this.

## Tools used and why (summary)

- **Make.com**: automation platform, free tier. Chosen for its visual error-handling model and native connectors for this specific combination of apps. See full reasoning above.
- **Tally**: form intake, free tier with genuine webhook support. See full reasoning above.
- **Airtable**: CRM substitute, free tier. See full reasoning above.
- **Hunter.io**: domain enrichment, free tier (50 lookups/month). See full reasoning above.
- **Gemini (`gemini-3.5-flash-lite`)**: scoring engine, free tier for testing, paid-tier pricing used in cost calculations above. See full reasoning above.
- **Slack**: real-time alerting for Hot leads only, using a bot connection rather than a user connection, so the integration is not tied to any one person's individual account. Free tier.
