# Validation Results — checkout-api v2.4

Evidence gathered locally on 2026-08-20 from a clean clone of `feature/readiness-pack`, plus reference to the repo's own CI pipeline.

## 1. CI pipeline (`.github/workflows/ci.yml`)

Steps it runs on every push/PR to main: checkout, setup Python 3.11, `pip install -r app/requirements.txt`, `python -m py_compile app/app.py app/test_app.py` (lint/syntax gate), `pytest app/test_app.py`, then a Docker Buildx dry-run build (`push: false`) tagged `checkout-api:v2.4`.

Recent runs on this repo (via `gh run list --repo kalviumcommunity/release-readiness-lab`) show the pipeline passing consistently, ~25-30s runtime:

```
completed  success  CI Pipeline  feature/readiness-pack  pull_request  32101926349  28s  2026-08-18T05:11:33Z
completed  success  CI Pipeline  feature/readiness-pack  pull_request  32099460154  27s  2026-08-18T04:31:29Z
completed  success  CI Pipeline  checks                  pull_request  31147296481  28s  2026-08-07T04:24:03Z
```

This branch's own CI run will show up under Actions once pushed — check there for the run tied to this exact commit.

## 2. Local dependency install

```
$ cd app && python -m venv venv && source venv/Scripts/activate
$ pip install -r requirements.txt
...
Successfully installed Flask-3.0.2 Jinja2-3.1.6 MarkupSafe-3.0.3 Werkzeug-3.1.8 blinker-1.9.0
click-8.4.2 colorama-0.4.6 gunicorn-22.0.0 iniconfig-2.3.0 itsdangerous-2.2.0 packaging-26.3
pluggy-1.6.0 pytest-8.0.2
```

All three pinned deps installed clean, no resolver conflicts. Note: local dev Python was 3.14.3 (machine default), not the 3.11 the Dockerfile and CI target — this only matters for the venv-based checks below; the actual container build uses `python:3.11-slim` per the Dockerfile, so runtime behavior in prod is unaffected by the local Python version.

## 3. Syntax / lint check (mirrors CI step)

```
$ python -m py_compile app.py test_app.py
$ echo $?
0
```
No syntax errors.

## 4. Unit tests

```
$ pytest test_app.py -v
============================= test session starts =============================
platform win32 -- Python 3.14.3, pytest-8.0.2, pluggy-1.6.0
collected 2 items

test_app.py::test_index PASSED                                           [ 50%]
test_app.py::test_health PASSED                                          [100%]

============================== 2 passed in 0.29s ==============================
```
Both tests pass. Note both tests only assert `status_code == 200` and the presence of keys in the health check — neither test asserts the actual value of `checks.database` / `checks.redis`, so CI would pass even in the fallback state below. That's a test-coverage gap worth flagging (see risk-analysis.md).

## 5. Live run — confirms the config gap first-hand

Ran the Flask app directly (`python app.py`) with no env vars set, then hit both endpoints:

```
$ curl -s http://localhost:5000/health
{"checks":{"database":"warning_fallback_sqlite","redis":"warning_fallback_local"},"status":"healthy","version":"v2.4"}

$ curl -s http://localhost:5000/
{"message":"Release Readiness Demo","service":"checkout-api","version":"v2.4"}
```

`/health` still reports `"status": "healthy"` with HTTP 200 even while both dependencies are in their volatile fallback mode — a readiness/liveness probe watching this endpoint would see the pod as healthy right through a state-loss condition. This is the same env-var gap called out in `config-verification.md`.

## 6. Container build

Docker CLI is present (`docker --version` → 29.2.0) but Docker Desktop's daemon was not running on this machine, so `docker build -t checkout-api:v2.4 .` could not be executed locally:

```
$ docker build -t checkout-api:v2.4 .
ERROR: failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine: ...daemon is running
```

Container build validation for this candidate therefore relies on the CI pipeline's Buildx dry-run step (which does execute the real build against the Dockerfile, just without pushing) — see the CI run evidence in section 1. Recommend confirming a fresh local build succeeds before any real cluster rollout, since this Dockerfile itself was never build-tested outside CI.

## 7. Dependency vulnerability scan

Ran `pip-audit` against the installed venv — see full output and findings in `dependency-checks.md`.

## 8. Kubernetes manifest syntax

No live cluster was available (`kubectl version --client` → v1.34.1, but no cluster context configured — `kubectl apply --dry-run=client` failed trying to reach `localhost:8080`). Fell back to a structural YAML check (parses each manifest, confirms `apiVersion`/`kind`/`metadata` present):

```
k8s/configmap.yaml  -> valid YAML, kind= ConfigMap
k8s/deployment.yaml -> valid YAML, kind= Deployment
k8s/service.yaml    -> valid YAML, kind= Service
ALL_MANIFESTS_SYNTAX_OK
```

This confirms the manifests are syntactically well-formed, not that they'll apply cleanly against a real cluster (RBAC, existing namespace, CRDs, etc. are unverified). See `rollback-plan.md` for what "tested" means here given that constraint.
