# Client Onboarding Automation

A Zapier automation that handles the operational steps required when a new client signs up: creating a client record, sending the service agreement for signature, adding the client to the email list, and notifying the internal team.

## Problem

New client signup involves several small, easy to forget administrative tasks: logging the client, getting a signed agreement in place, adding them to ongoing communications, and letting the right people know a new client has come in. Handled manually, these steps are prone to delay and inconsistency, particularly when volume increases or the person handling onboarding is occupied elsewhere. This automation runs the full sequence the moment a new client is confirmed, with no manual intervention required.

## Trigger

The automation is designed against the payload structure of a Stripe payment confirmation event (`payment_intent.succeeded`), capturing the client's email, name, and selected plan at the moment payment is received. Stripe account access was not available in the environment this was built in, so the trigger was implemented using Webhooks by Zapier configured to accept a payload matching Stripe's documented event structure. Swapping in a live Stripe connection requires only reselecting the trigger app; every downstream step is already built against the same field names Stripe's webhook provides.

## Pipeline

1. **New Payment Received**: captures the incoming payment event and the client's details.
2. **Create Client Record**: logs the client, plan, and signup date to an Airtable base.
3. **Send Onboarding Contract**: sends a service agreement for signature via a DocuSign template, populated with the client's name, plan, and effective date.
4. **Add Client to Email List**: creates or updates the client's contact record in Brevo and adds them to the onboarding email list.
5. **Notify Team in Slack**: posts a concise notification to an internal Slack channel confirming the new client and plan.

## Tool selection

**Zapier** over Make or n8n: this pipeline is a linear sequence of standard business actions with no conditional branching, retrieval, or custom computation. Each of the four platforms could execute this flow, but Zapier's native app catalog covers every tool in this stack (Airtable, DocuSign, Brevo, Slack) without a single custom HTTP call, which keeps the build fast and the maintenance surface small. Reaching for a more complex orchestration platform here would add operational overhead without adding capability. This project intentionally demonstrates that judgment: matching the tool to the actual shape of the problem rather than defaulting to the most capable platform available.

**DocuSign** over a simpler document tool: a service agreement carries legal weight, so a genuine e-signature flow with a defined signer role and audit trail is used rather than a static document or an unsigned PDF.

**Brevo** over a larger marketing platform: at the client volumes this system is built for, Brevo's contact management and free tier support the onboarding email need without the cost of a heavier CRM-adjacent tool.

## Scope decisions

- **No conditional logic on plan tier.** Every plan follows the same onboarding sequence. Differentiated onboarding paths per plan tier were left out because the current plan structure does not yet warrant different treatment; this is revisited if plan tiers diverge enough to need separate contract terms or communication tracks.
- **No retry or error-handling branches.** Unlike the multi-source, multi-stage pipeline in the account monitoring system, this pipeline is a single linear sequence against stable, well-supported native integrations. Zapier's built-in step-level error notifications are considered sufficient at this scale; custom retry logic was judged to add complexity without a corresponding reduction in real failure risk for a four-step linear flow.
- **No duplicate-signup handling.** A single payment event maps to a single onboarding run. Deduplication logic was scoped out because Stripe's own payment event structure does not produce duplicate confirmations for a single successful charge under normal operation.

## Enterprise scaling

- Swapping the webhook-based trigger for a live Stripe connection is a single-step change; no downstream step requires modification since all four actions already read from the same field structure Stripe's webhook provides.
- Each action step (Airtable, DocuSign, Brevo, Slack) can be replaced independently if the underlying tool changes, without requiring changes to the other three steps.
- The linear structure is intentionally simple to extend: adding a step (for example, a calendar invite for an onboarding call) is a matter of inserting one additional action into the existing sequence.

## Cost

Airtable, Brevo, and Webhooks by Zapier are used on free tiers at this project's scale. DocuSign requires a paid plan for production use beyond its trial allowance; Slack is used on a free workspace tier. A production deployment handling meaningful client volume would need to budget for DocuSign's per-envelope or per-seat pricing and, depending on volume, a Zapier plan tier above the free tier for higher monthly task limits.

