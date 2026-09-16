# Day 11 — Scheduling & High Availability

## Checklist
- [ ] NodeSelector
- [ ] Node Affinity
- [ ] Taints & Tolerations
- [ ] Pod Affinity
- [ ] Pod Anti-Affinity
- [ ] Topology Spread Constraints
- [ ] Multi-node / multi-AZ thinking
- [ ] Ensure replicas are distributed across failure domains
- [ ] Explain scheduling tradeoffs

---

## 1. NodeSelector — Simplest Placement Constraint

```yaml
spec:
  nodeSelector:
    disktype: ssd        # Pod ONLY schedules on nodes labeled disktype=ssd
```

- Exact-match only, no operators, no "preferred" option — either a node matches or the Pod stays Pending.

## 2. Node Affinity — Expressive NodeSelector

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # hard rule
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution:  # soft rule (best-effort)
      - weight: 80
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a"]
```

- `required...` = hard constraint (must match, or Pod stays Pending).
- `preferred...` = soft constraint (scheduler tries, but will still place the Pod elsewhere if needed), weighted 1-100 to rank among multiple soft rules.
- `IgnoredDuringExecution` means: if node labels change after the Pod is running, the Pod is NOT evicted — affinity is only checked at scheduling time.

## 3. Taints & Tolerations — Repel, Not Attract

```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

- **Taints** live on **nodes** and repel Pods by default ("nothing may schedule here unless it tolerates this taint").
- **Tolerations** live on **Pods** and allow (not force) scheduling onto a tainted node.
- Effects: `NoSchedule` (won't schedule new Pods), `PreferNoSchedule` (soft), `NoExecute` (evicts already-running Pods that don't tolerate it — used for node cordoning/draining scenarios and `node.kubernetes.io/not-ready` handling).

**Key distinction from affinity:** affinity/nodeSelector is about a Pod **choosing** a node it wants; taints/tolerations are about a node **rejecting** Pods it doesn't want (e.g., reserving GPU nodes only for ML workloads, keeping system-critical nodes free of general workloads).

## 4. Pod Affinity / Anti-Affinity — Relative to Other Pods

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - {key: app, operator: In, values: ["api"]}
        topologyKey: kubernetes.io/hostname   # don't co-locate replicas on same node
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - {key: app, operator: In, values: ["cache"]}
          topologyKey: kubernetes.io/hostname  # co-locate with cache pods for low latency
```

- **Pod anti-affinity**: spread replicas apart (different nodes/zones) — critical for **availability** (a single node/zone failure doesn't take out all replicas).
- **Pod affinity**: pull related Pods together (e.g., app + local cache sidecar-like pattern across pods) — useful for **latency** optimization.
- `topologyKey` defines the "domain" of spreading: `kubernetes.io/hostname` (per-node), `topology.kubernetes.io/zone` (per-AZ).

## 5. Topology Spread Constraints — Even Distribution at Scale

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway (soft)
    labelSelector:
      matchLabels: {app: api}
```

- More flexible/scalable than pod anti-affinity for large replica counts — instead of "never on the same node," it says "keep the max difference in Pod count between any two zones/nodes to `maxSkew`."
- `whenUnsatisfiable: DoNotSchedule` = hard constraint; `ScheduleAnyway` = best-effort.
- This is the modern, preferred way to achieve balanced multi-AZ distribution (anti-affinity historically has scheduling performance issues at high replica counts).

## 6. Multi-AZ / Failure-Domain Design

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector: {matchLabels: {app: api}}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector: {matchLabels: {app: api}}
              topologyKey: kubernetes.io/hostname
```

- With 6 replicas across 3 AZs and `maxSkew: 1`, the scheduler aims for 2 per AZ — losing one entire AZ still leaves 4/6 replicas running.
- Combine with a **PodDisruptionBudget** so voluntary disruptions (node drains, upgrades) never take availability below a safe threshold:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 4       # out of 6 replicas
  selector: {matchLabels: {app: api}}
```

## 7. Scheduling Tradeoffs (Interview Framing)

| Tradeoff | Explanation |
|----------|--------------|
| Strict (`required`) vs soft (`preferred`) rules | Strict rules guarantee placement behavior but risk Pods stuck `Pending` if unsatisfiable; soft rules always schedule but may violate your ideal distribution under pressure |
| Anti-affinity vs topology spread | Anti-affinity is precise but scales poorly (O(n²) comparisons) with many replicas; topology spread is more efficient and is the modern default for HA fan-out |
| Bin-packing vs spreading | Spreading replicas maximizes availability but can increase cost (more nodes lightly used); bin-packing maximizes node utilization/cost-efficiency but concentrates failure risk |
| Taints for isolation | Great for dedicating expensive nodes (GPU) to specific workloads, but adds operational overhead (must remember tolerations, can cause Pending pods if misconfigured) |

## 8. Hands-On Lab

```bash
kubectl label node <node1> zone=a --overwrite
kubectl label node <node2> zone=b --overwrite
kubectl label node <node3> zone=c --overwrite

kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-demo
spec:
  replicas: 6
  selector: {matchLabels: {app: spread-demo}}
  template:
    metadata: {labels: {app: spread-demo}}
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector: {matchLabels: {app: spread-demo}}
      containers:
      - name: nginx
        image: nginx:1.27-alpine
EOF

kubectl get pods -l app=spread-demo -o wide
# Verify even distribution across zone=a/b/c
```

## 9. Interview Points
- "Affinity is a Pod expressing a preference; taints/tolerations are a node expressing a restriction — opposite directions of the same mechanism."
- "For HA, I use topology spread constraints across `topology.kubernetes.io/zone` combined with a PodDisruptionBudget, so I guarantee both even distribution and a minimum surviving replica count during voluntary disruptions."
- "Required rules give guarantees but risk unschedulable Pods; I use preferred rules when best-effort placement is acceptable, required when correctness truly depends on it (e.g., regulatory data locality)."
