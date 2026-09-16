# Day 3 — API & Controllers

## Checklist
- [ ] Kubernetes API objects
- [ ] Reconciliation loop
- [ ] Desired state vs actual state
- [ ] How controllers maintain desired state
- [ ] What happens internally when a Deployment is created?
- [ ] Understand etcd failure and API Server failure at a high level
- [ ] Explain the scheduler's responsibility

---

## 1. Kubernetes API Objects

Every object in Kubernetes (Pod, Deployment, Service, ConfigMap...) shares the same structure:

```yaml
apiVersion: apps/v1     # API group/version
kind: Deployment        # Object type
metadata:                # Name, namespace, labels, annotations, UID
  name: my-app
spec:                    # DESIRED state (you write this)
  replicas: 3
status:                  # ACTUAL/observed state (Kubernetes writes this)
  readyReplicas: 3
```

- `spec` = what you want. `status` = what the cluster observes right now. Controllers exist purely to close the gap between the two.
- Objects are organized into **API groups** (`apps`, `networking.k8s.io`, `rbac.authorization.k8s.io`, core `""`) and versions (`v1`, `v1beta1`).

```bash
kubectl api-resources          # list all object types
kubectl explain deployment.spec.strategy
kubectl explain deployment.spec.strategy.rollingUpdate --recursive
```

## 2. The Reconciliation Loop

```
        ┌─────────────────────────────────────────┐
        │              Control Loop                │
        │                                          │
   ┌────▼─────┐    compare     ┌────────────┐      │
   │  Watch    │───────────────▶│  Desired   │      │
   │  actual   │                │  (spec)    │      │
   │  state    │◀───────────────│  vs Actual │      │
   └────┬─────┘     diff?       │  (status)  │      │
        │                       └────────────┘      │
        │ yes → take action to converge             │
        └─────────────────────────────────────────┘
              (repeats forever, event-driven + periodic resync)
```

This is the **core design pattern of all of Kubernetes** — every controller (built-in or custom Operator) is just:
```
for {
  observed := getCurrentState()
  desired  := getDesiredState()
  if observed != desired {
     takeAction()
  }
}
```

## 3. How Controllers Maintain Desired State — Example: Deployment

When you create a `Deployment`:

1. API Server persists the `Deployment` object to etcd.
2. **Deployment controller** (in kube-controller-manager) watches for Deployment objects. It sees a new one with no matching ReplicaSet → creates a **ReplicaSet** with the Pod template and desired `replicas`.
3. **ReplicaSet controller** watches for ReplicaSets. It sees `replicas: 3` but 0 matching Pods exist → creates 3 **Pod** objects (unscheduled, no `nodeName`).
4. **Scheduler** watches for Pods with no `nodeName` → assigns each to a node.
5. **kubelet** on each assigned node watches for Pods bound to it → starts the containers via the container runtime.
6. kubelet reports Pod status back → ReplicaSet controller sees `readyReplicas` climb → once it matches `replicas`, the loop is quiet (until something changes again: a pod dies, someone edits the Deployment, a node fails, etc.)

Every arrow above is an independent **watch + reconcile** loop — no component polls in a tight loop; they use the API Server's **watch** (long-lived HTTP streaming) mechanism to react instantly to changes.

## 4. etcd Failure vs API Server Failure

| Failure | Impact |
|---------|--------|
| **etcd down** (all replicas) | Cluster state cannot be read or written. API Server returns errors on any request that needs etcd. Existing Pods **keep running** (kubelets don't need etcd directly), but nothing can be scheduled, scaled, healed, or changed. This is the most catastrophic failure — always run 3 or 5 etcd members and back it up. |
| **API Server down** (all replicas) | No one (kubectl, controllers, kubelets) can read/write cluster state. Running Pods **keep running** on nodes (kubelet caches last known state and containers keep executing), but no new scheduling, no status updates, no self-healing happens until API Server is back. |
| **Single etcd/API server replica down (HA setup)** | No visible impact — other replicas behind the load balancer serve traffic; Raft leader election in etcd handles it transparently. |

**Key takeaway for interviews:** the data plane (already-running Pods, kube-proxy rules, existing networking) is resilient to short control-plane outages, but the cluster loses its "self-healing brain" until the control plane recovers. This is why control-plane HA (3+ replicas across AZs) is non-negotiable in production.

## 5. Scheduler's Responsibility

The **Scheduler** does NOT run anything — it only **decides where** a Pod should run, then writes that decision (a "binding") back to the API Server.

Two-phase process:
1. **Filtering (predicates):** eliminate nodes that can't run the Pod — insufficient CPU/memory (based on `requests`), taints without matching tolerations, node selectors/affinity not matched, port conflicts, volume zone mismatches.
2. **Scoring (priorities):** rank remaining nodes — spread pods evenly, prefer nodes with fewer resources already allocated (or bin-packing depending on policy), honor pod affinity/anti-affinity, topology spread constraints.
3. Highest-scoring node wins → Scheduler updates the Pod's `spec.nodeName` via the API Server.
4. The kubelet on that node then takes over actually starting the containers.

**Interview soundbite:** "The scheduler is a matchmaking service — it never talks to nodes directly and never starts containers. It just writes an assignment decision to etcd via the API server; kubelet does the actual work."

## 6. Hands-On

```bash
# Watch objects change in real time across the reconciliation chain
kubectl get deployments,rs,pods -l app=aspnet-app -w &
kubectl apply -f ../day-02-pods-deployments/aspnet-deployment.yaml

# Inspect the ownership chain via ownerReferences
kubectl get rs -o jsonpath='{.items[0].metadata.ownerReferences}'
kubectl get pods -o jsonpath='{.items[0].metadata.ownerReferences}'

# See scheduler decisions in events
kubectl get events --field-selector reason=Scheduled

# Explore full object status vs spec
kubectl get deployment aspnet-app -o json | jq '.spec, .status'
```
