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

## Best practices used in this chart
- **Run as non-root and drop privileges**:
  - `podSecurityContext.runAsNonRoot: true`, `runAsUser: 1000`, `runAsGroup: 1000`
  - `securityContext.allowPrivilegeEscalation: false`
  - `securityContext.readOnlyRootFilesystem: true`
  - `securityContext.capabilities.drop: ["ALL"]`
  - `podSecurityContext.seccompProfile.type: RuntimeDefault`

- **Dedicated ServiceAccount with minimal exposure**:
  - `serviceAccount.create: true` by default, with `serviceAccount.name` override
  - Pods set `serviceAccountName` accordingly
  - `automountServiceAccountToken: false` on Pods to avoid mounting tokens unless needed

- **Immutable and reproducible images**:
  - Default `image.pullPolicy: IfNotPresent`
  - Recommend pinning `image.digest` (e.g., `sha256:...`) instead of mutable tags for production

- **Health probes for fast recovery**:
  - Readiness and liveness probes enabled and configurable; optional startup probe
  - Keep probe thresholds/timeouts aligned with app startup/steady-state behavior

- **Resource requests/limits and autoscaling**:
  - Sensible default requests/limits to protect cluster SLOs
  - HPA enabled with CPU-based scaling (`minReplicas`, `maxReplicas`, `targetCPUUtilizationPercentage`)

- **Disruption tolerance**:
  - PodDisruptionBudget enabled by default (`minAvailable: 2`) to limit voluntary disruptions

- **Predictable scheduling and high availability**:
  - `podAntiAffinity: required` to spread replicas across nodes
  - `topologySpreadConstraints` to evenly distribute Pods (requires ≥3 nodes for `replicaCount: 3`)
  - If your cluster has fewer nodes, set `podAntiAffinity: preferred`

- **Network security by default**:
  - Optional NetworkPolicies
  - Default deny egress policy, plus explicit DNS egress allow (TCP/UDP 53)
  - Extra egress rules can be provided via `networkPolicy.additionalEgressRules`

- **Ingress and traffic management**:
  - Optional Ingress with class selection (`ingress.className`)
  - Canary releases via Argo Rollouts with stable/canary Services and NGINX traffic routing

- **Safe rollouts (progressive delivery)**:
  - Argo Rollouts canary steps: gradual weights and pauses by default
  - Optional `rollout.analysis` (templates/args) for metric-based gating

- **Consistent naming and labels**:
  - Uses Helm helpers for names; no hard-coded resource names
  - Standard Kubernetes labels under `app.kubernetes.io/*` for better tooling and queries

- **Operational guidance**:
  - Prefer pinning image digests for supply-chain integrity
  - Consider adding Prometheus scrape annotations if you expose metrics
  - Keep Secrets out of images; use secret stores (e.g., External Secrets) if needed

These defaults aim to be secure and production-lean while remaining easy to override through `values.yaml` for your environment.

## Argo CD Application example
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: httpbin
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/lizardDroid/appdesign
    targetRevision: HEAD
    path: deployments/httpbin
    helm:
      releaseName: httpbin
      values: |
        ingress:
          enabled: true
          className: nginx
          hosts:
            - host: httpbin.example.com
              paths:
                - path: /
                  pathType: Prefix
        rollout:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
