# Automations

A working portfolio of business automation systems, built end to end: intake, AI-driven decision logic, error handling, and audit-grade logging. Each project is built to the standard of something a client would actually run in production, not a linear demo of connecting two apps, and each is designed with an eye toward what changes if the volume or the client is ten times bigger.

The tools change per project based on what the problem actually calls for: Make.com and n8n for orchestration requiring conditional logic and custom computation, Zapier where a linear sequence of well-supported native integrations is the right fit for the problem's actual complexity, whichever LLM API fits the cost and reasoning requirements of the task, and direct REST integration wherever a prebuilt connector doesn't exist or doesn't go far enough. Every project ships with a full architecture writeup that covers not just what was built, but how it would need to change to handle a solo founder's ticket volume versus a client running thousands of transactions a day.

## Capabilities demonstrated across this portfolio

- Multi-branch conditional automation (routers, filters, error handlers) across Make and n8n, designed to extend cleanly rather than require a rebuild as volume grows
- Direct REST/HTTP API integration, with and without prebuilt connectors
- LLM integration for classification, scoring, and generation tasks, provider chosen per project on cost and output quality, not defaulted to one vendor
- Structured data handling across nested API responses (JSON parsing, field mapping)
- Retrieval-augmented generation: vector embedding, similarity search, and context-grounded scoring against a structured knowledge base
- Failure-path design: independent error handling per external call, so a single failed API call doesn't take down the rest of the process. The same principle applies whether a system processes ten transactions a day or ten thousand
- Cost modeling and tool-selection tradeoffs, documented per project, including where the architecture would need to change at higher scale (database over spreadsheet, queued processing over synchronous calls, multi-tenant data isolation)
- Platform judgment: matching the orchestration tool to the actual shape of the problem, rather than defaulting to the most capable platform available regardless of fit

## Projects

| Project | Platform | What it does |
|---|---|---|
| [Support Triage Engine](./support-triage) | Make.com | Classifies inbound support tickets by category and urgency, drafts a suggested reply, and routes each one by AI confidence score, with duplicate detection and independent failure paths on every external call. |
| [Lead Qualification & CRM Sync](./lead-qualification) | Make.com | Enriches inbound leads via domain lookup, scores them against a hybrid deterministic and AI model, and syncs qualified leads into a CRM layer with tiered routing for high-priority leads. |
| [Ops Reporting Engine](./ops-reporting-engine) | n8n | Detects revenue anomalies against a trailing statistical baseline, generates a plain-language explanation of each flagged anomaly, and delivers a daily report via Slack and email. |
| [Signal Intelligence Engine](./account-signal-monitoring) | n8n | Monitors multiple external sources for business development signals, scores each one against a structured ideal customer profile using retrieval-augmented generation, and routes qualified opportunities to the appropriate channel in near real time. |
| [Client Onboarding Automation](./client%20onboarding%20automation) | Zapier | Runs the full onboarding sequence for a new client on payment confirmation: creates a client record, sends the service agreement for signature, adds the client to the email list, and notifies the internal team. |

Each project folder contains a full README covering the problem it solves, its architecture, tool-selection rationale, error handling, deliberate scope decisions, enterprise-scaling considerations, and cost.
