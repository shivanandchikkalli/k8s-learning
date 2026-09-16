# Day 10 — Scaling

## Checklist
- [ ] Horizontal Pod Autoscaler (HPA)
- [ ] CPU/memory based scaling
- [ ] Metrics Server
- [ ] Custom metrics concept
- [ ] Cluster Autoscaler
- [ ] HPA vs Cluster Autoscaler
- [ ] Vertical Pod Autoscaler concept
- [ ] Queue-driven scaling architecture
- [ ] Practice scaling a workload

---

## 1. Horizontal Pod Autoscaler (HPA)

- Adjusts the **number of Pod replicas** in a Deployment/StatefulSet based on observed metrics vs a target.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 70}
  - type: Resource
    resource:
      name: memory
      target: {type: Utilization, averageUtilization: 80}
```

- HPA polls metrics every 15s (default), computes `desiredReplicas = ceil(currentReplicas * currentMetric / targetMetric)`, and updates the Deployment's `replicas`.

## 2. Metrics Server — Prerequisite for CPU/Memory HPA

- **Metrics Server** is a lightweight, in-cluster component that collects real-time CPU/memory usage from kubelets (via the `metrics.k8s.io` API) and exposes it for `kubectl top` and HPA.
- Without it installed, CPU/memory-based HPA **cannot function** — HPA has nothing to read.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top nodes
kubectl top pods
```

## 3. Custom Metrics — Beyond CPU/Memory

- Real-world autoscaling often needs to react to things CPU/memory can't capture: requests-per-second, queue depth, latency.
- Requires a **custom/external metrics adapter** (e.g., Prometheus Adapter) that exposes metrics through the `custom.metrics.k8s.io` or `external.metrics.k8s.io` API so HPA can consume them just like CPU/memory.

```yaml
metrics:
- type: Pods
  pods:
    metric: {name: http_requests_per_second}
    target: {type: AverageValue, averageValue: "100"}
```

## 4. Cluster Autoscaler — Scaling Nodes, Not Pods

| | HPA | Cluster Autoscaler |
|---|-----|---------------------|
| Scales | Number of **Pod replicas** | Number of **nodes** in the cluster |
| Trigger | CPU/memory/custom metric vs target | Pods stuck **Pending** (unschedulable due to insufficient node capacity) OR nodes sitting idle/underutilized |
| Acts on | Deployment/StatefulSet | Node Groups / Auto Scaling Groups (cloud) |

**How they work together:** HPA scales Pods up → if there's no room on existing nodes, new Pods go `Pending` → Cluster Autoscaler notices unschedulable Pods → adds nodes → scheduler places the Pods. When load drops, HPA scales Pods down → Cluster Autoscaler notices underutilized nodes → drains and removes them.

```
Load increases
     │
     ▼
HPA adds more Pod replicas
     │
     ▼
Not enough node capacity? → Pods stay Pending
     │
     ▼
Cluster Autoscaler adds new Nodes
     │
     ▼
Scheduler places the Pending Pods on new Nodes
```

## 5. Vertical Pod Autoscaler (VPA) — Concept

- Instead of adding more Pods, VPA adjusts a Pod's **requests/limits** (CPU/memory) based on historical usage — "right-sizing" existing Pods.
- Modes: `Off` (recommendation only), `Initial` (set at creation only), `Auto`/`Recreate` (evicts and recreates Pods with new values).
- **Do not combine VPA and HPA on the same metric** (e.g., both reacting to CPU) — they can fight each other. Common pattern: VPA for memory right-sizing, HPA for CPU-based horizontal scaling, or VPA in recommendation-only mode.

## 6. Queue-Driven Scaling Architecture

For asynchronous/worker workloads (batch processing, order fulfillment, document processing), scaling on CPU is often the wrong signal — a worker can be CPU-idle while waiting on I/O yet have a huge backlog.

```
┌────────────┐     ┌───────────────┐     ┌──────────────────┐
│  Producers  │────▶│  Queue (SQS,  │────▶│  Worker Deployment │
│ (API, jobs) │     │  RabbitMQ,    │     │  (KEDA / custom    │
└────────────┘     │  Kafka)       │     │   metrics adapter) │
                    └───────┬───────┘     └─────────┬─────────┘
                            │ queue depth metric      │
                            └─────────────────────────┘
                             HPA/KEDA scales replicas
                             based on messages-per-pod
```

```yaml
# KEDA ScaledObject example
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-worker
spec:
  scaleTargetRef: {name: order-worker}
  minReplicaCount: 0        # can scale to zero when queue is empty
  maxReplicaCount: 50
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/orders
      queueLength: "5"       # target ~5 messages per worker pod
```

- This lets workers scale purely on **backlog size**, including scaling to **zero** when idle — impossible with plain CPU-based HPA.

## 7. Hands-On Lab

```bash
kubectl create deployment php-apache --image=registry.k8s.io/hpa-example
kubectl set resources deployment php-apache --requests=cpu=200m
kubectl expose deployment php-apache --port=80

kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa -w &

kubectl run load-generator --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# Watch replicas climb, then stop load and watch scale-down (~5 min stabilization)
kubectl delete pod load-generator
```

## 8. Interview Points
- "HPA scales Pods based on metrics; Cluster Autoscaler scales Nodes based on unschedulable Pods — they're complementary, not competing."
- "Metrics Server is a hard prerequisite for CPU/memory HPA; for anything else I need a custom/external metrics adapter."
- "For queue-based workers, I scale on backlog depth (via KEDA), not CPU, because CPU usage doesn't reflect actual backlog pressure — and I can scale to zero when idle."
