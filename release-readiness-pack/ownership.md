# Deployment Ownership — checkout-api v2.4

No ownership or escalation path existed anywhere in the repo before this pack. Roles below are lab placeholders (this is a training exercise, not a real org) — in a real deployment these would be actual names/handles, not role labels.

| Role | Responsibility | Placeholder |
|---|---|---|
| Release Owner | Owns the go/no-go call, the readiness pack, and coordinates the deploy window. Final authority on whether v2.4 ships. | release-owner@checkout-team (placeholder) |
| SRE / Operational Lead | Owns the k8s manifests, config/secrets correctness, rollback execution, and cluster health during/after the deploy | sre-lead@checkout-team (placeholder) |
| On-Call (deploy window) | First responder for any alert/incident during and for 24h after the deploy window | on-call rotation, see internal PagerDuty schedule (placeholder — no real rotation configured for this lab) |
| Escalation | If on-call can't resolve within 30 min, escalate to SRE Lead; if still unresolved within 60 min, escalate to Release Owner + trigger rollback per rollback-plan.md | n/a (lab) |

## Deploy window

Not yet scheduled — blocked on the No-Go call in `readiness-report.md`. Once the blocking risks (rows 1-2 in `risk-analysis.md`) are closed, the Release Owner schedules the window and confirms On-Call coverage before proceeding.

## Handoff

Post-deploy, On-Call owns monitoring for the first 24h. Any rollback decision during that window can be made unilaterally by On-Call per the trigger criteria in `rollback-plan.md`, without waiting for Release Owner sign-off — the criteria are pre-approved so response isn't blocked on availability.
