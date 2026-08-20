# Dependency Checks — checkout-api v2.4

## Pinning

`app/requirements.txt`:
```
Flask==3.0.2
gunicorn==22.0.0
pytest==8.0.2
```

All three are exact-pinned with `==`, not ranges or unpinned — this is already good practice, nothing to fix here. Matches `docs/architecture.md`'s stated requirements (Flask 3.0.2, Gunicorn 22.0.0, Python 3.11).

## Installed versions (confirmed via clean venv install)

```
$ pip install -r requirements.txt
Successfully installed Flask-3.0.2 Jinja2-3.1.6 MarkupSafe-3.0.3 Werkzeug-3.1.8 blinker-1.9.0
click-8.4.2 colorama-0.4.6 gunicorn-22.0.0 iniconfig-2.3.0 itsdangerous-2.2.0 packaging-26.3
pluggy-1.6.0 pytest-8.0.2
```

Exactly the pinned versions resolved, no drift.

## Vulnerability scan

Ran `pip-audit` against the installed venv:

```
$ pip install pip-audit
$ pip-audit
Found 7 known vulnerabilities in 3 packages
Name   Version ID              Fix Versions
------ ------- --------------- ------------
flask  3.0.2   PYSEC-2026-2151 3.1.3
pip    25.3    PYSEC-2026-196  26.1.2
pip    25.3    PYSEC-2026-1796 26.0
pip    25.3    PYSEC-2026-196  26.1.2
pip    25.3    PYSEC-2026-2875 26.1
pip    25.3    PYSEC-2026-2876 26.1
pytest 8.0.2   PYSEC-2026-1845 9.0.3
```

Breakdown:
- **Flask 3.0.2** — `PYSEC-2026-2151`, fixed in 3.1.3. This ships in the production container (it's an app runtime dependency), so this is the one that actually matters for the deployment decision.
- **pytest 8.0.2** — `PYSEC-2026-1845`, fixed in 9.0.3. Test-only dependency, not shipped in the production image (the Dockerfile only `COPY`s `app.py`, and `requirements.txt` installs pytest into the image regardless — see note below). Lower operational risk but should still be bumped.
- **pip 25.3** (4 advisories) — this is the packaging tool itself, not an app dependency; it's whatever pip version ships with the base Python image / venv, not something pinned in `requirements.txt`. Irrelevant to the running container's attack surface but worth patching the build environment.

Note: the Dockerfile installs the full `requirements.txt` — including `pytest` — into the production image (`RUN pip install --no-cache-dir -r requirements.txt`), then only copies `app.py`. That means `pytest` and its transitive deps end up in the shipped image with no test files to run against them — unnecessary attack surface and image bloat. Recommend splitting into `requirements.txt` (Flask, gunicorn) and `requirements-dev.txt` (pytest), and only installing the dev file in CI, not the Dockerfile.

## Gunicorn worker config

Not a version issue, but adjacent: `Dockerfile`'s `CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]` doesn't set `--workers`/`--threads`, so it runs Gunicorn's default of 1 sync worker per pod. At 3 replicas that's 3 total concurrent request-handling processes for a "critical" checkout path. Recommend explicit `--workers` (commonly `2 * CPU + 1`) tuned to the pod's CPU limit (currently `500m` in `deployment.yaml`).

## Image tag

`k8s/deployment.yaml` pins `image: kalvium/checkout-api:v2.4` with `imagePullPolicy: IfNotPresent` — a mutable tag, not a digest. `IfNotPresent` means a node that already cached an older `v2.4` layer (from a bad build) won't pull the new one. Recommend pinning by digest (`@sha256:...`) for reproducible rollouts and rollbacks, or at minimum switching to `imagePullPolicy: Always` if tag-based deploys are kept.

## Verdict

Pinning: pass. Vulnerability posture: **needs action** — bump Flask to 3.1.3 before this ships (see risk-analysis.md, row 3). pytest bump and dev/prod requirements split are lower-priority follow-ups, not release blockers on their own.
