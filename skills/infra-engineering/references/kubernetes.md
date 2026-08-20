# Kubernetes

## Workload resource choice

| Resource | Use for |
|---|---|
| `Deployment` | Stateless apps; supports rolling updates, easy scale up/down. Default choice. |
| `StatefulSet` | Anything needing stable network identity and/or persistent storage per replica (databases, queues) — stable pod names (`app-0`, `app-1`), ordered start/stop, per-replica PVCs via `volumeClaimTemplates`. |
| `DaemonSet` | Exactly one pod per node (log shippers, node monitoring agents, CNI/CSI components). |
| `Job` / `CronJob` | Run-to-completion tasks / scheduled run-to-completion tasks. Set `backoffLimit` and `activeDeadlineSeconds` so a broken job doesn't retry forever or hang indefinitely. |

Don't reach for `StatefulSet` out of caution for an app that's actually stateless — it complicates rollout and scaling for no benefit.

## Pod spec essentials

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        fsGroup: 10001
      containers:
        - name: api
          image: registry.example.com/api:1.4.2   # never :latest
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits: { memory: 512Mi }   # no CPU limit — see below
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 15
            periodSeconds: 20
          ports: [{ containerPort: 8080 }]
```

## Readiness vs. liveness vs. startup probes

- **Readiness**: is this pod ready to receive traffic *right now*. Failing readiness pulls the pod out of Service endpoints without killing it — the correct response to "temporarily overloaded" or "dependency unreachable."
- **Liveness**: is this process alive/unstuck. Failing liveness kills and restarts the container. A liveness probe that checks downstream dependencies (DB, other services) causes cascading restarts when the dependency — not the app — is down. Keep liveness checks to "is my own process responsive," not "is everything I depend on healthy."
- **Startup**: for slow-starting apps, gates liveness/readiness until startup succeeds so a normal slow boot isn't mistaken for a liveness failure and killed mid-startup. Use instead of a huge `initialDelaySeconds` on liveness.

## Resource requests/limits

- Always set `requests` — the scheduler uses this to place pods; unset means the scheduler treats it as effectively zero, which leads to node overcommit and noisy-neighbor evictions.
- Set memory `limits` (OOM is a clean, recoverable failure). Generally *don't* set CPU `limits` — Kubernetes CPU limits are enforced via hard throttling (CFS quota) even when the node has idle CPU, which causes latency spikes under bursty load that are hard to diagnose. Bound CPU with `requests` for scheduling and let it burst; use `LimitRange`/`ResourceQuota` at the namespace level if you need a hard ceiling for cost/fairness reasons instead.
- `QoS` class follows from these: `requests == limits` on everything → `Guaranteed` (best eviction protection); requests set, limits unset/higher → `Burstable`; neither set → `BestEffort` (first to be evicted under node pressure).

## Services and networking

| Type | Use for |
|---|---|
| `ClusterIP` (default) | Internal-only access — the default for anything not meant to be reached from outside the cluster. |
| `NodePort` | Rarely the right final answer; mainly for on-prem/dev without a cloud load balancer. |
| `LoadBalancer` | Cloud-provisioned external LB, one per Service — expensive at scale, and every one is a public entry point that needs to be deliberate, not a default. |
| `Ingress` | HTTP(S) routing/TLS termination for many Services behind one LB. Needs an ingress controller (nginx, Traefic, cloud-native ALB controller, etc.) installed in the cluster. |

- `NetworkPolicy` resources are how you actually restrict which pods can talk to which — without any `NetworkPolicy`, all pod-to-pod traffic in the cluster is allowed by default regardless of namespace. Default-deny then allow specific paths for anything handling sensitive data:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata: { name: default-deny }
  spec:
    podSelector: {}
    policyTypes: [Ingress, Egress]
  ```
  Requires a CNI that enforces NetworkPolicy (Calico, Cilium; the default `kubenet` on some setups does not).

## ConfigMaps and Secrets

- `ConfigMap` for non-sensitive config, mounted as env vars or files.
- `Secret` for sensitive values — but a raw `Secret` manifest is only base64-encoded, not encrypted; anyone with API read access to that namespace (or the etcd backing store, if it isn't encrypted at rest) can read it in plaintext. Enable etcd encryption at rest at the cluster level, and for anything committed to a repo use **Sealed Secrets**, **External Secrets Operator** (syncs from Vault/AWS Secrets Manager/etc.), or **SOPS** — never a bare `Secret` YAML with real values in git.
- Mounting a `Secret` as a volume updates automatically on change (with a propagation delay); as env vars it does not — the pod must be restarted to pick up new values. This affects rotation design.

## RBAC

Least privilege, scoped to namespace by default:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: app, name: pod-reader }
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { namespace: app, name: pod-reader-binding }
subjects:
  - kind: ServiceAccount
    name: app-sa
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
```
`ClusterRole`/`ClusterRoleBinding` only when access genuinely spans namespaces or targets cluster-scoped resources (nodes, PVs, CRDs). `cluster-admin` bound to anything but a human break-glass account or the cluster's own control-plane components is a standing red flag. Every pod gets a dedicated `ServiceAccount` with `automountServiceAccountToken: false` unless it actually needs to call the API server — the `default` service account should not be used for workloads.

## Autoscaling

- **HPA** (Horizontal Pod Autoscaler) scales replica count on a metric (CPU/memory by default, custom/external metrics via an adapter for queue depth, request rate, etc.). Requires `requests` to be set — HPA's default CPU-utilization target is a percentage of the request.
- **VPA** (Vertical Pod Autoscaler) adjusts requests/limits instead of replica count — don't run HPA-on-CPU and VPA on the same workload simultaneously without care, they can fight each other.
- **Cluster Autoscaler** (or Karpenter) adds/removes nodes based on unschedulable pods / node utilization — a separate layer from pod-level autoscaling; HPA can scale pods past what current nodes can fit, and cluster autoscaling is what makes that actually land.

## Helm

- `helm template`/`helm diff upgrade` (via the `helm-diff` plugin) before `helm upgrade` against anything real — see exactly what will change first.
- Values files layered by environment (`values.yaml` base + `values-prod.yaml` override via `-f`), not one giant file with commented-out sections.
- Pin chart versions explicitly (`helm install ... --version 1.4.2`); don't float on whatever's latest in the repo index.
- `helm rollback <release> <revision>` is the fast path back — know the current revision (`helm history <release>`) before upgrading so rollback is a known-good target, not a guess.

## Troubleshooting

Diagnosis order that works:

1. `kubectl get pods -o wide` — is it even scheduled? `Pending` → check `kubectl describe pod` events for scheduling failures (resource pressure, node affinity/taint mismatch, PVC not bound).
2. `kubectl describe pod <pod>` — Events section at the bottom explains most crashes: image pull errors, probe failures, OOMKilled, volume mount failures.
3. `kubectl logs <pod> [-c <container>] [--previous]` — `--previous` is essential for a pod that's already restarted; the current logs are from the new (possibly not-yet-failed) instance.
4. `CrashLoopBackOff` → almost always an app-level startup failure (check `logs --previous`) or a misconfigured probe killing it before it's ready — not a Kubernetes problem to fix at the YAML level unless the probe timing is the actual cause.
5. `ImagePullBackOff` → wrong image name/tag, missing `imagePullSecrets` for a private registry, or registry auth expired.
6. `OOMKilled` (check `kubectl describe pod`, reason field) → memory limit too low for actual usage, or a real leak — check the trend across restarts, not just one data point.
7. Service not routing → confirm `Service` selector matches pod labels (`kubectl get endpoints <svc>`; empty endpoints means the selector matches nothing), then check `NetworkPolicy` if endpoints are populated but traffic still doesn't land.

## Common failure modes

- **Rolling update takes down all replicas briefly**: `maxUnavailable` too high relative to replica count, or no `PodDisruptionBudget` protecting against voluntary disruption (node drains, cluster autoscaler) on top of the deployment's own rollout settings.
- **Pod evicted under node pressure**: `BestEffort`/`Burstable` QoS with no memory `limits`, or a `Guaranteed` neighbor squeezing it out. Set requests/limits deliberately (see above).
- **Config change doesn't take effect**: `ConfigMap`/`Secret` mounted as env vars requires a pod restart to pick up new values — rolling the deployment (`kubectl rollout restart deployment/<name>`) or bumping a checksum annotation in the pod template (common Helm pattern) forces it.
- **`kubectl apply` conflicts with GitOps/Helm-managed resources**: manual `apply`/`edit` gets reverted on the next sync, or causes a `helm upgrade` to fail on ownership conflicts. Change the source (Helm values, Git-managed manifest) instead of the live object.
