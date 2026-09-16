# Day 6 — Resource Management

## Checklist
- [ ] CPU requests
- [ ] Memory requests
- [ ] CPU limits
- [ ] Memory limits
- [ ] QoS classes
- [ ] OOMKilled
- [ ] CPU throttling
- [ ] Understand scheduling based on requests
- [ ] Practice resource limits on a workload

---

## 1. Requests vs Limits

```yaml
resources:
  requests:       # GUARANTEED minimum — used by the SCHEDULER to place the pod
    cpu: "250m"
    memory: "256Mi"
  limits:         # HARD CEILING — enforced by the kubelet/runtime at runtime
    cpu: "500m"
    memory: "256Mi"
```

- **Requests** answer: "How much does the scheduler reserve for me on a node?" The scheduler only places a Pod on a node that has enough **unallocated requested** capacity — it never looks at limits for scheduling decisions.
- **Limits** answer: "What's the max this container can consume before Kubernetes intervenes?"
- `1 CPU` = 1 vCPU/core. `500m` = 0.5 CPU (millicores). Memory: `Mi` (mebibytes), `Gi` (gibibytes).

## 2. What Happens When You Exceed Requests/Limits

| Resource | Exceeding request (limit not hit) | Exceeding limit |
|----------|-----------------------------------|------------------|
| **CPU** | Fine — CPU is compressible; container can burst using spare node capacity | **Throttled** (not killed) — Linux CFS quota mechanism delays/limits CPU time, causing latency spikes |
| **Memory** | Fine — memory is compressible only up to a point | **OOMKilled** — memory is incompressible; kernel OOM killer terminates the container (exit code 137), kubelet restarts it per `restartPolicy` |

```bash
kubectl describe pod <pod>
# Look for: Last State: Terminated, Reason: OOMKilled, Exit Code: 137
```

## 3. CPU Throttling — Why It's Sneaky

- CPU limits are enforced in fixed time windows (default 100ms CFS periods). If a container bursts above its limit even briefly, it gets throttled for the rest of that window — this shows up as **latency spikes**, not crashes, so it's easy to miss without proper metrics (`container_cpu_cfs_throttled_seconds_total`).
- **Common architect debate:** many teams set memory limits (must, to protect node stability) but deliberately **omit CPU limits** (only set requests) to avoid invisible throttling, relying on the scheduler + requests for fairness instead.

## 4. Quality of Service (QoS) Classes

Kubernetes automatically assigns a QoS class per Pod based on requests/limits — this determines **eviction priority** when a node is under memory pressure.

| QoS Class | Condition | Eviction priority |
|-----------|-----------|-------------------|
| **Guaranteed** | Every container has `requests == limits` for BOTH cpu and memory | Evicted **last** — most protected |
| **Burstable** | At least one container has requests set, but requests ≠ limits (or limits missing) | Evicted **second** |
| **BestEffort** | No requests or limits set at all | Evicted **first** — least protected |

```bash
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
```

**Never run production workloads as BestEffort** — they're the first to be killed under any node memory pressure, with zero guarantees.

## 5. Scheduling Based on Requests

- Scheduler sums up all **requests** (not limits, not actual usage) of Pods already placed on each node, and only considers a node "fits" if `node.allocatable - sum(requests) >= new pod's requests`.
- This means a node can be **overcommitted** on limits (sum of limits > node capacity) but never overcommitted on requests for scheduling purposes.
- If requests are set too high → wasted capacity (nodes look "full" but actual usage is low). If requests are too low → **scheduler over-packs the node**, and real usage spikes cause throttling/evictions.

```bash
kubectl describe node <node> | grep -A 8 "Allocated resources"
```

## 6. Hands-On Lab

```yaml
# file: resource-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: polinux/stress
    resources:
      requests: {cpu: "200m", memory: "100Mi"}
      limits:   {cpu: "200m", memory: "100Mi"}    # requests == limits => Guaranteed
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
---
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx:1.27-alpine    # no resources at all => BestEffort
```

```bash
kubectl apply -f resource-demo.yaml

# Watch the Guaranteed pod get OOMKilled (requests 150M into a 100Mi limit)
kubectl get pod guaranteed-pod -w
kubectl describe pod guaranteed-pod | grep -A5 "Last State"

# Compare QoS classes
kubectl get pod guaranteed-pod -o jsonpath='{.status.qosClass}'; echo
kubectl get pod besteffort-pod -o jsonpath='{.status.qosClass}'; echo

kubectl delete -f resource-demo.yaml
```

## 7. Interview Points
- "Requests drive scheduling; limits drive runtime enforcement — they answer different questions."
- "Memory is incompressible, so exceeding a memory limit gets you OOMKilled; CPU is compressible, so exceeding a CPU limit just gets you throttled."
- "I always set memory requests=limits to get Guaranteed QoS for critical workloads, and I'm cautious with CPU limits because they can cause invisible throttling under bursty load."
