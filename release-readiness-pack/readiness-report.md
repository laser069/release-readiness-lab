# Release Readiness Report — checkout-api v2.4

## Decision: **No-Go**

v2.4 is not ready for production. Two release-blocking risks, both confirmed with first-hand evidence in this pack, not hypothetical.

## Reasoning

**Blocking risk 1 — missing DATABASE_URL/REDIS_URL (risk-analysis.md, row 1).** Confirmed live: running the app with no env vars set, `/health` returns `200 healthy` while both dependencies report `warning_fallback_*`. At the deployment's 3 replicas, this means each pod holds private, non-shared, non-persistent state — checkout data would diverge per-pod and vanish on every restart. Nothing in the current probe setup would catch this, because `/health` reports healthy regardless (risk-analysis.md, row 3). This is a data-integrity issue for a checkout service, not a cosmetic gap.

**Blocking risk 2 — Flask 3.0.2 has a known, confirmed vulnerability.** `pip-audit` run against the actual installed dependency set found `PYSEC-2026-2151` in Flask 3.0.2, fixed in 3.1.3 (dependency-checks.md). This ships in the production container as-is.

Everything else checked out reasonably well: dependencies are properly pinned (`==`), the CI pipeline runs and passes consistently (validation-results.md), unit tests pass, the k8s manifests are syntactically valid and structurally sound (namespace, labels, probes, resource limits all present and consistent), and a rollback target (v2.3, last STABLE per docs/architecture.md) is unambiguous.

## What must change to reach Go

1. Add `DATABASE_URL` and `REDIS_URL` to the deployment via a Kubernetes `Secret` (not the plaintext ConfigMap) — config-verification.md.
2. Bump `Flask==3.1.3` in `requirements.txt`, re-run the test suite, rebuild the image — dependency-checks.md.
3. Re-verify: confirm `/health` reports `"database": "connected"` and `"redis": "connected"` against real Postgres/Redis instances, and re-run `pip-audit` clean on Flask.

Once those two are closed, recommend also closing before the *next* release window (not blocking, but should not linger): make `/health` fail (non-200) when either dependency is degraded, add a `Secret`-based pattern for future config, and execute the rollback plan once against a real staging cluster rather than leaving it command-reviewed-only (risk-analysis.md rows 3, 5, 6).

## Evidence index

- `validation-results.md` — CI reference, local test run, live `/health` proof of the config gap, container/manifest validation and what couldn't be run locally
- `config-verification.md` — env var audit vs. code and docs
- `dependency-checks.md` — pinning check + `pip-audit` scan results
- `rollback-plan.md` — target version, commands, verification steps, what's tested vs. reviewed-only
- `risk-analysis.md` — full risk matrix, 10 rows, severity/likelihood/mitigation/owner
- `ownership.md` — roles and escalation path

## Sign-off

No-Go as of 2026-08-20. Re-review required after the two blocking items above are closed — this report should be re-run (or explicitly updated) against the fixed build before any deploy proceeds.
