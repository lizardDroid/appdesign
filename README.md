# httpbin-helm

Minimal, secure-by-default Helm chart to deploy **kennethreitz/httpbin** on a 3-node Kubernetes cluster with **even Pod distribution**, canary rollout via **Argo Rollouts**, and a clean verification flow.

## Key features
- Even spread: `topologySpreadConstraints` + hard `podAntiAffinity` (1 pod per node on a 3-node cluster).
- Canary: Argo Rollouts with NGINX traffic splitting (20% → 50% → 100% by default).
- Security: non-root, read-only root FS, no privilege escalation, drop all caps, `seccompProfile: RuntimeDefault`.
- Reliability: probes, PDB (minAvailable=2), conservative rolling updates.
- Autoscaling-ready: HPA (CPU target) enabled by default.
- Minimal by design: opt-in Ingress/NetworkPolicy, no RBAC beyond a dedicated, permissionless SA.

## Quick start (classic verification)
```bash
# 1) Create namespace
kubectl create namespace web || true

# 2) Install
helm upgrade --install httpbin ./deployments/httpbin -n web

# 3) Check rollout (Argo Rollouts)
kubectl -n web get rollout httpbin

# 4) Check even distribution (pods on different nodes)
kubectl -n web get pods -l app.kubernetes.io/name=httpbin -o wide

# 5) Verify HTTP 200 (port-forward to stable service)
kubectl -n web port-forward svc/httpbin 8080:80 &
sleep 2
curl -i http://127.0.0.1:8080/
```
