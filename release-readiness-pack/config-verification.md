# Configuration Verification — checkout-api v2.4

Compared what `app/app.py` and `docs/architecture.md` say the app needs against what `k8s/configmap.yaml` actually provides.

## Env vars the app reads

| Var | Read in code | Documented in architecture.md | Present in configmap.yaml | Effect if missing |
|---|---|---|---|---|
| `PORT` | `app.py` (`os.getenv("PORT", 5000)`) | yes | yes (`"5000"`) | none, has a safe default |
| `LOG_LEVEL` | `app.py` (`os.getenv("LOG_LEVEL", "INFO")`) | yes | yes (`"info"`) | none, has a safe default |
| `DATABASE_URL` | `app.py` `/health` handler | yes, required for PostgreSQL connection | **no** | app falls back to in-memory SQLite (`db_status = "warning_fallback_sqlite"`) |
| `REDIS_URL` | `app.py` `/health` handler | yes, required for Redis cache | **no** | app falls back to a local in-memory dict cache (`redis_status = "warning_fallback_local"`) |

`k8s/configmap.yaml` even has a code comment admitting this:
```
# Note: Developers forgot to include DATABASE_URL and REDIS_URL here.
# This causes the application to default to volatile, in-memory SQLite and local cache.
```

Confirmed live (see `validation-results.md` section 5): running the app with no env vars set, `/health` returns `200 healthy` with both checks in fallback mode. The probe doesn't fail, so Kubernetes has no signal that the deployment is degraded.

## Why this matters at 3 replicas

`k8s/deployment.yaml` runs `replicas: 3`. If `DATABASE_URL`/`REDIS_URL` stay unset, each of the 3 pods gets its own private in-memory SQLite file and its own private in-memory cache dict — no shared state across replicas, and all state is lost on every pod restart/reschedule. For a checkout service this means cart/order data silently diverging per-pod and vanishing on redeploy. This is the single biggest readiness gap found in this review — see risk-analysis.md, row 1.

## Secrets handling

Even once `DATABASE_URL` and `REDIS_URL` are added, they should not go into the plaintext `ConfigMap` as currently structured — connection strings for Postgres/Redis normally embed credentials. Recommend a Kubernetes `Secret` (or an external secrets manager synced via `envFrom.secretRef`) for these two specifically, keeping `PORT`/`LOG_LEVEL` in the existing ConfigMap. No `Secret` manifest exists anywhere in this repo currently.

## What's correct as-is

- `PORT` and `LOG_LEVEL` are both present, both have safe code-level defaults regardless.
- Namespace (`checkout-system`) is consistent across `configmap.yaml`, `deployment.yaml`, and `service.yaml`.
- `envFrom.configMapRef` wiring in `deployment.yaml` correctly points at `checkout-api-config` — the pattern is right, only the data is incomplete.

## Verdict

Config verification: **fails**. `DATABASE_URL` and `REDIS_URL` must be added (as a Secret, not the ConfigMap) before this is production-safe. See readiness-report.md for how this gates the go/no-go call.
