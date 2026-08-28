# Automated Revenue Anomaly Detection & Reporting

A daily reporting pipeline that reads sales data, detects statistically significant revenue anomalies, identifies which product category drove each anomaly, and delivers an explanation to the operations team through Slack and email.

## Problem

Manually checking a sales dashboard every day doesn't scale, and it's easy to miss a real anomaly on the day it happens. This system runs on a schedule, flags only the days that are statistically unusual, and explains the likely cause using that day's category-level breakdown. It identifies not just that revenue moved, but which product line drove it.

## How it works

**Data ingestion.** A scheduled trigger reads daily sales data (date, product category, units sold, revenue) from a connected spreadsheet.

**Anomaly detection.** A Python step aggregates the raw rows into daily totals and per-category breakdowns, then calculates a trailing 14-day rolling average and standard deviation. A z-score is computed against that baseline, and any day with an absolute z-score of 2 or higher is flagged as an anomaly. Two weeks of history are required before a day can be evaluated.

**Root cause explanation.** Flagged days are passed to a call that reasons over that day's category breakdown and explains which category drove the anomaly, whether it was a spike or a drop, and the likely business cause. The category breakdown is carried through the pipeline specifically so this explanation is grounded in real data rather than the aggregate number alone.

**Delivery.** Each anomaly triggers a Slack alert for immediate visibility and a styled HTML email for reference. Normal days are not reported; the system stays silent when nothing needs attention.

**Failure handling.** The data read and the explanation call each have independent error handlers that post a specific failure notice to the operations channel. If the explanation call fails after an anomaly is already detected, the notice still includes the raw anomaly numbers so the team isn't left with nothing.

## Why these choices

**n8n over Make.** Anomaly detection needs real statistical computation, specifically a rolling average and standard deviation. n8n's native Code node runs this as plain Python at no extra cost; Make's chained-formula approach would be more fragile and harder to audit for the same calculation.

**Deterministic detection, generative explanation.** The anomaly threshold itself is a fixed z-score calculation, not a model call, because it needs to be consistent and auditable. The model is only used after a day is flagged, to interpret the breakdown and explain it in plain language.

**Category-level data, not just the aggregate.** An earlier version discarded the per-category split after computing the daily total, which meant the explanation step had nothing to reason from and produced generic, unverifiable guesses. Carrying the breakdown through fixed that.

**Two delivery channels.** Slack for immediate visibility, email for a reference copy with the full detail. Each channel is used for what it's actually good at.

## Scope

This system explains anomalies after detection; it doesn't forecast them or recommend action. Forecasting is a different kind of model and needs its own validation before it belongs in a production alert. It was left out deliberately, not because it would be hard to bolt on. The explanation gives enough context for a person to judge quickly whether a spike is good news or a drop needs a response; automating that judgment was left to the person receiving the alert.

## Cost and scaling

At current volume, ongoing cost is the per-call cost of the explanation model, incurred only on days an anomaly is actually detected, plus a negligible fixed cost for delivery. Model pricing should be confirmed against current published rates before use in a client-facing estimate.

Detection and explanation logic scale without structural changes at higher volume, since each day's calculation is independent. The part that would need rework at meaningfully higher scale is data ingestion, which currently reads from a single spreadsheet. A higher-volume deployment would read from a database or data warehouse instead.
