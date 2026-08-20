# Rollback Plan — checkout-api v2.4

## Rollback target

Per `docs/architecture.md` release history, **v2.3** is the last version marked `Deployed (Prod - STABLE)`. v2.4 is only a Release Candidate. If v2.4 fails post-deploy, roll back to v2.3.

| Version | Status |
|---|---|
| v2.3 | Deployed (Prod - STABLE) — **rollback target** |
| v2.4 | Release Candidate — current deploy under review |

## Rollback commands

Two ways, pick based on how the deploy was rolled out:

**1. Kubernetes-native rollback** (if v2.4 was rolled out via `kubectl apply`/`kubectl set image` and revision history is intact):
```bash
kubectl rollout history deployment/checkout-api -n checkout-system
kubectl rollout undo deployment/checkout-api -n checkout-system
kubectl rollout status deployment/checkout-api -n checkout-system --timeout=120s
```

**2. Explicit image pin** (safer if revision history is missing or ambiguous — makes the rollback target unambiguous rather than relying on "previous" revision):
```bash
kubectl set image deployment/checkout-api checkout-api=kalvium/checkout-api:v2.3 -n checkout-system
kubectl rollout status deployment/checkout-api -n checkout-system --timeout=120s
```

Recommend (2) as the primary method for this deployment specifically, given the mutable-tag issue flagged in dependency-checks.md — "previous revision" via `rollout undo` is ambiguous if the same tag was ever re-pushed.

## Post-rollback verification

```bash
kubectl get pods -n checkout-system -l app=checkout-api -o wide
kubectl get deployment checkout-api -n checkout-system -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl exec -n checkout-system deploy/checkout-api -- curl -s http://localhost:5000/
kubectl exec -n checkout-system deploy/checkout-api -- curl -s http://localhost:5000/health
```
Confirm: all 3 pods `Running` and `READY 1/1`, image field reads `kalvium/checkout-api:v2.3`, `/` reports `"version": "v2.3"`, `/health` reports `"status": "healthy"`.

## What was actually verified vs. what needs a live cluster

No live Kubernetes cluster was available for this review (see validation-results.md section 8) — kubectl client is installed (v1.34.1) but no cluster context is configured on this machine. What was verified:

- Manifests are syntactically valid YAML with correct `apiVersion`/`kind`/`metadata` (checked programmatically, see validation-results.md).
- The rollback commands above are standard, well-documented kubectl operations correct for this Deployment's structure (single container, `checkout-api` name, `checkout-system` namespace — all confirmed against `k8s/deployment.yaml`).

What was **not** verified end-to-end: an actual `kubectl rollout undo`/`set image` executed against a running cluster with a real v2.4 pod, followed by confirming traffic actually serves v2.3. That requires a cluster (kind/minikube/staging) this environment doesn't have. Recommend running this exact sequence against a staging cluster before relying on it in a real incident — treat the commands above as reviewed-correct, not incident-tested.

## Rollback trigger criteria

Roll back if, after v2.4 deploy:
- `/health` reports non-200 or `status != "healthy"` for any pod for more than 2 consecutive probe intervals, or
- error rate on `/` or checkout-related endpoints exceeds baseline, or
- `DATABASE_URL`/`REDIS_URL` are confirmed still unset in the live ConfigMap/Secret post-deploy (i.e. the config gap in config-verification.md shipped anyway) — treat this as an automatic rollback trigger, not a "monitor and see."
