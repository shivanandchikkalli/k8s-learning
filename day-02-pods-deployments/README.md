# Day 2 — Pods, ReplicaSets & Deployments

## Checklist
- [ ] Pod lifecycle and restart behavior
- [ ] Deployment → ReplicaSet → Pods
- [ ] Create an ASP.NET Core Deployment
- [ ] Scale replicas up/down
- [ ] Rolling update and rollout history
- [ ] Roll back a deployment
- [ ] Interview: What happens when a Pod or node dies?

---

## 1. Pod Lifecycle

```
Pending → Running → Succeeded / Failed
              │
              └─ container restarts happen INSIDE Running, per restartPolicy
```

| Phase | Meaning |
|-------|---------|
| Pending | Accepted by cluster, not yet scheduled or image still pulling |
| Running | Bound to a node, at least one container running |
| Succeeded | All containers exited with 0 (Jobs) |
| Failed | At least one container exited non-zero and won't restart |
| Unknown | Node unreachable |

**restartPolicy**: `Always` (default, Deployments), `OnFailure` (Jobs), `Never`.

Container restart uses **exponential backoff** (`CrashLoopBackOff`): 10s, 20s, 40s ... capped at 5 min.

## 2. Ownership Chain

```
Deployment  (manages rollout strategy & history)
    │  owns
    ▼
ReplicaSet  (ensures N identical pod replicas exist)
    │  owns
    ▼
Pod x N
```

- You almost never create ReplicaSets or Pods directly in production — you create a **Deployment**, and it creates/manages ReplicaSets, which create/manage Pods.
- Each rollout (image/config change) creates a **new ReplicaSet**; the old one is scaled to 0 but kept (for rollback) up to `revisionHistoryLimit`.

## 3. Hands-On: ASP.NET Core Deployment

```yaml
# file: aspnet-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aspnet-app
  labels:
    app: aspnet-app
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: aspnet-app
  template:
    metadata:
      labels:
        app: aspnet-app
        version: v1
    spec:
      containers:
      - name: aspnet-app
        image: mcr.microsoft.com/dotnet/samples:aspnetapp
        ports:
        - containerPort: 8080
        env:
        - name: ASPNETCORE_URLS
          value: "http://+:8080"
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            memory: "256Mi"
```

```bash
kubectl apply -f aspnet-deployment.yaml
kubectl get deployments
kubectl get rs                 # ReplicaSet created by the Deployment
kubectl get pods -o wide       # Pods created by the ReplicaSet
kubectl describe deployment aspnet-app
```

## 4. Scaling

```bash
# Imperative
kubectl scale deployment aspnet-app --replicas=5
kubectl get pods -w

# Declarative (preferred): edit replicas in YAML, re-apply
kubectl apply -f aspnet-deployment.yaml

# Scale down
kubectl scale deployment aspnet-app --replicas=2
```

## 5. Rolling Update & Rollback

```bash
# Trigger a rolling update by changing the image
kubectl set image deployment/aspnet-app aspnet-app=mcr.microsoft.com/dotnet/samples:aspnetapp-8.0

# Watch the rollout
kubectl rollout status deployment/aspnet-app

# See revision history
kubectl rollout history deployment/aspnet-app
kubectl rollout history deployment/aspnet-app --revision=2

# Roll back to previous version
kubectl rollout undo deployment/aspnet-app

# Roll back to a specific revision
kubectl rollout undo deployment/aspnet-app --to-revision=1
```

### RollingUpdate strategy fields

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # extra pods allowed above desired count during update
      maxUnavailable: 0    # how many desired pods can be unavailable during update
```

- `maxUnavailable: 0` + `maxSurge: 1` → zero-downtime updates (new pod up and ready before an old one is removed).

## 6. Interview: What happens when a Pod or node dies?

**Pod dies (container crashes, OOMKilled, etc.):**
1. kubelet on the node detects the container exited.
2. Based on `restartPolicy` (default `Always` for Deployment pods), kubelet restarts the **container in place** (same Pod, same node, same IP is NOT guaranteed to persist — actually Pod IP stays the same as long as the Pod object itself isn't deleted).
3. If it keeps crashing, kubelet backs off exponentially → `CrashLoopBackOff` status.
4. The Pod itself is unchanged; only the container inside restarts.

**Pod is deleted entirely, or node dies:**
1. If a Pod is deleted (or evicted) and it's managed by a ReplicaSet, the **ReplicaSet controller** notices actual replica count < desired count and creates a **brand-new Pod** (new name, new IP) to compensate.
2. If a **node** dies: the **Node controller** waits for a grace period (`node-monitor-grace-period`, default ~40s) marking the node `NotReady`, then after `pod-eviction-timeout` (default 5 min) it evicts/marks Pods on that node for deletion.
3. The ReplicaSet controller then schedules replacement Pods on healthy nodes.
4. Any Service pointing at the old Pod's IP automatically stops routing to it (Endpoints are updated) and starts routing to the new Pod once it's Ready.

**Key point:** Kubernetes never "heals" a Pod — it always replaces it. Pods are ephemeral/disposable by design; only the controller's desired state (replica count) is durable.
