# AI-Powered Support Ticket Triage

A Make.com automation that classifies incoming support tickets, drafts a suggested reply, and routes each one to the right channel based on how confident the classification is, with duplicate detection and error handling built in throughout.

## The problem

Consider a small SaaS company, two or three people, no dedicated support team. Every ticket that arrives has to be read by a person before anything can happen with it: what category is this, how urgent is it, who should handle it. At low volume that overhead is invisible. Once a team is fielding 30 or 40 tickets a day, triage becomes its own job, and it's usually the founder or whoever happens to be free absorbing that time, sorting tickets rather than resolving them.

This system automates that specific bottleneck. It doesn't attempt full customer support automation. It handles the triage decision that currently consumes time without producing much value on its own.

## How it works

**Intake.** A Google Form stands in for whatever channel a real business would use, a contact form, an embedded widget, a forwarded email address. The intake mechanism is interchangeable; the logic that follows is what does the work.

**Duplicate detection.** Before any AI call runs, the system checks whether the submitting email has a recent ticket on record, and whether it's actually the same issue. A second submission in the same category is treated as a duplicate and routed to an alert instead of being reprocessed. A submission in a different category is treated as a new ticket, since it likely is one. The one ambiguous case is a ticket categorized as "Other," which carries little information on its own. For that category specifically, the system falls back to a short time window, roughly ten minutes, and only flags a repeat "Other" submission as a duplicate within that window. Past it, the submission is treated as genuinely new.

**Classification.** A Gemini API call reads the ticket and returns a category, an urgency level, and a confidence score. The prompt requires the model to state its reasoning before producing the confidence score, which turned out to matter more than expected. An earlier version of this build used a lighter Gemini model that returned near-maximum confidence regardless of how ambiguous the ticket actually was, even after the prompt explicitly defined what each confidence band should mean. The issue wasn't the prompt; it was that the lighter model wasn't built to reason carefully under uncertainty. Moving to a stronger model and restructuring the prompt so reasoning precedes the score resolved the calibration problem.

**Audit logging.** Every ticket that reaches this point is written to a Google Sheet with a timestamp, contact details, and classification, independent of what happens next. This exists because a Slack channel is not a system of record. Messages get missed, muted, or scrolled past. A business relying on this needs a durable place to confirm what came in and how it was handled.

**Reply drafting.** A second Gemini call generates a suggested reply in a professional tone. It's surfaced in Slack as a draft for a human to review and send, never sent automatically. This is a deliberate design decision rather than a missing feature. Unreviewed AI text reaching a customer directly carries real risk, incorrect tone, factual errors, invented policy detail, and a review step is worth the added friction.

**Confidence-based routing.** Tickets scoring 0.7 or higher route to the primary support channel. Anything below that threshold routes to a separate review channel, since a low-confidence classification shouldn't be treated the same as a clear one. A human makes the final call on the ambiguous cases.

**Error handling.** If the classification call fails, whether from an invalid key, a rate limit, or an outage, a dedicated fallback still alerts a human so the ticket isn't silently dropped. If the reply-drafting call fails specifically, a separate alert notes that the ticket was classified successfully but needs a manually written reply. That failure path does not reach the audit sheet, which is a genuine constraint of how Make handles branching logic rather than an oversight, and it's documented here rather than left for someone else to discover.

## Deliberate scope limitations

- **No SLA escalation clock.** Nothing currently tracks whether an urgent ticket has gone unaddressed for too long. Left out to keep this build focused on getting classification, routing, and error handling right before expanding scope.
- **Single intake channel.** Only form submissions are handled, not email. This mirrors a decision many small businesses actually make, routing all support requests through one form specifically because it forces structured fields, a defined category, a clear description, rather than parsing unstructured email text. Multi-channel intake is a reasonable next step, but a single, deliberately structured entry point is a legitimate design choice in its own right.
- **Fixed duplicate window.** The ten-minute window used for "Other"-category tickets is a flat constant rather than configurable. Reasonable as a default; a production version would likely expose it as a setting.

## Cost and return on investment

This build runs on `gemini-3.5-flash`, priced at $1.50 per million input tokens and $9.00 per million output tokens as of August 2026.

Each ticket triggers two AI calls, classification and reply drafting:

| Call | Input tokens | Output tokens |
|---|---|---|
| Classification | ~500 | ~150 |
| Reply drafting | ~600 | ~200 |
| **Total per ticket** | **~1,100** | **~350** |

That comes to approximately **$0.005 per ticket**, under half a cent.

| Tickets per month | Estimated AI cost |
|---|---|
| 500 | ~$2.40 |
| 5,000 | ~$24 |
| 50,000 | ~$240 |

For comparison, manual triage at roughly 2 minutes per ticket and a $15/hour wage costs approximately $0.50 per ticket, over a hundred times the AI cost. At 5,000 tickets a month, that's approximately $2,500 in labor spent purely on sorting, against $24 in API spend. This system isn't a substitute for a support representative; it removes the sorting overhead so that time goes toward resolving tickets rather than categorizing them.

The model choice is worth explaining. Gemini Flash-Lite, at roughly five to ten times lower cost per token, was tested first and rejected. It failed at the one requirement that mattered most for this use case, producing a trustworthy confidence score, returning near-certainty on tickets a human would immediately flag as ambiguous. At the volume a small team would realistically see, the cost difference between the two models amounts to a few dollars a month. That difference isn't worth trading away the core function the system depends on.

This estimate covers Gemini API cost only. It does not include Make.com platform costs beyond the free-tier operation limit, which is a separate, volume-dependent expense outside the scope of this analysis.

## Built with

Make.com for orchestration, Google Forms for intake, the Gemini API called directly over HTTP with no prebuilt connector, Google Sheets as the audit log, and Slack for alerts and routing via a bot OAuth connection.

## Why these tools, and why this scales

None of the tools in this build are load-bearing in the sense that the architecture depends on them specifically. Each was chosen for a reason, and each has a clear substitution path if a client or employer's stack calls for something else.

**Make.com over Zapier or n8n.** Make was chosen for this project because its visual branching (routers, filters, per-module error handlers) maps directly onto the decision logic this system needs, confidence-based routing, duplicate detection, and independent failure paths for two separate AI calls, without writing custom code to express that branching. Zapier can do simpler linear automations but handles multi-branch conditional logic less directly. n8n offers the same branching model as Make with the added option of native code nodes, which makes it a natural next step for a version of this same system that needs custom logic Make's built-in modules can't express. The architecture itself, trigger, dedup check, classify, log, draft, route, error handle, transfers to n8n with minimal change, since the decision logic doesn't depend on Make specifically.

**Gemini over other model providers.** Gemini was chosen primarily on cost, its per-token pricing is meaningfully lower than comparable-tier models from other providers, which matters directly at scale: this system makes two AI calls per ticket, so at 50,000 tickets a month that cost difference compounds. The classification and reply-drafting logic here isn't tied to Gemini's API specifically. Both calls are plain HTTP requests with a JSON body, the same pattern used to call OpenAI, Anthropic, or any other provider with a REST API. Swapping providers means changing an endpoint URL and an authentication header, not rebuilding the automation.

**Google Sheets over a proper database.** Sheets is the right choice at this scale because it's free, requires no setup, and is something any team member can open and read without technical tools. It stops being the right choice once ticket volume or query complexity grows past what a spreadsheet handles well. At that point the audit log module points to a proper database instead, Airtable as a low-friction step up, or Postgres/Supabase for a fully production setup, without changing anything upstream of that module. The rest of the pipeline doesn't know or care where the log lands.

**Slack over email or a dashboard.** Slack was chosen because it's where a small team is already working, an alert that requires opening a separate tool gets ignored. The same routing logic could post to Microsoft Teams, send email, or write to a ticketing system's API instead, again a matter of swapping the output module, not the decision logic that decides where a ticket goes.

**What scaling this up actually looks like.** The parts of this system that are genuinely simplified for a portfolio build, a flat duplicate window, no SLA clock, single intake channel, are exactly the parts called out above as deliberate scope decisions, not structural limits. Multi-channel intake means adding another trigger that feeds into the same dedup and classification logic. An SLA clock means a scheduled check against the audit log's timestamps. None of it requires re-architecting what's already built; it's addition, not replacement, which is the actual test of whether an automation was built to scale or just built to demo.
