# Automations

A working portfolio of business automation systems, built end to end: intake, AI-driven decision logic, error handling, and audit-grade logging. Each project is built to the standard of something a client would actually run in production, not a linear demo of connecting two apps, and each is designed with an eye toward what changes if the volume or the client is ten times bigger.

The tools change per project based on what the problem actually calls for, Make.com, n8n, and GoHighLevel for orchestration, whichever LLM API fits the cost and reasoning requirements of the task, and direct REST integration wherever a prebuilt connector doesn't exist or doesn't go far enough. Every project ships with a full architecture writeup that covers not just what was built, but how it would need to change to handle a solo founder's ticket volume versus a client running thousands of transactions a day.

## Capabilities demonstrated across this portfolio

- Multi-branch conditional automation (routers, filters, error handlers) across Make and n8n, designed to extend cleanly rather than require a rebuild as volume grows
- Direct REST/HTTP API integration, with and without prebuilt connectors
- LLM integration for classification, scoring, and generation tasks, provider chosen per project on cost and output quality, not defaulted to one vendor
- Structured data handling across nested API responses (JSON parsing, field mapping)
- Failure-path design: Independent error handling per external call, so a single failed API call doesn't take down the rest of the process. The same principle applies whether a system processes ten transactions a day or ten thousand.
- Cost modeling and tool-selection tradeoffs, documented per project, including where the architecture would need to change at higher scale (database over spreadsheet, queued processing over synchronous calls, multi-tenant data isolation)

## Projects

| Project | What it does |
|---|---|
| [Support Triage Engine](./support-triage/README.md) | Classifies inbound support tickets by category and urgency, drafts a suggested reply, and routes each one by AI confidence score, with duplicate detection and independent failure paths on every external call. |

More projects in progress. Each one is built for a specific scenario, but documented with the scaling path in mind, since the real test of an automation isn't whether it works once, it's whether it still works after the client's business grows past the version that got demoed.
