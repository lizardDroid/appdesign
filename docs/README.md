# Operations & Verification

## Install (canary on by default)
```bash
kubectl create namespace web || true
helm upgrade --install httpbin ./deployments/httpbin -n web
```

## Verify Argo Rollouts canary
```bash
kubectl -n web get rollout httpbin
kubectl -n web argo rollouts get rollout httpbin
# If you installed the kubectl plugin:
#   kubectl argo rollouts get rollout httpbin -n web
```

## Verify even distribution (1 pod per node on a 3-node cluster)
```bash
kubectl -n web get pods -l app.kubernetes.io/name=httpbin -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

## Health check
```bash
kubectl -n web port-forward svc/httpbin 8080:80 &
sleep 2
curl -i http://127.0.0.1:8080/
```

## Resilience (PDB) test
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl -n web get pods -w
```

## Canary notes
- Canary uses **NGINX** traffic routing managed by Argo Rollouts.
- Stable Ingress name: `httpbin` (if `ingress.enabled=true`).
- Steps: 20% → 50% → 100% (customizable in `values.yaml` under `rollout.steps`).

## Fallback to classic Deployment
```bash
helm upgrade --install httpbin ./deployments/httpbin -n web --set useDeploymentFallback=true
```
