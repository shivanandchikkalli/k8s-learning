# Day 15 — Staff-Level Kubernetes Architecture & Interview

## Checklist
- [ ] Design a 10-million-document processing system on Kubernetes
- [ ] Design a multi-tenant SaaS platform
- [ ] Design zero-downtime deployment
- [ ] Explain Rolling vs Blue/Green vs Canary
- [ ] Design queue-based worker scaling
- [ ] Explain multi-AZ high availability
- [ ] Explain resource isolation and quotas
- [ ] Explain security boundaries
- [ ] Troubleshoot a production latency incident
- [ ] Complete a mock Staff/Lead Kubernetes interview
- [ ] Explain Kubernetes architecture without notes

---

## 1. Design Exercise: 10-Million-Document Processing System

**Requirements framing:** documents arrive continuously, need parsing/OCR/indexing, must be durable, cost-efficient, and observable.

```
┌────────────┐   ┌───────────┐   ┌──────────────────┐   ┌───────────────┐
│  Upload API │──▶│  S3        │──▶│  SQS Queue        │──▶│ Worker Pods    │
│ (Deployment)│   │ (raw docs) │   │  (durable backlog) │   │ (KEDA-scaled,  │
└────────────┘   └───────────┘   └──────────────────┘   │  scale 0→N)    │
                                                            └───────┬───────┘
                                                                    ▼
                                                          ┌──────────────────┐
                                                          │ Result Store      │
                                                          │ (DB / OpenSearch) │
                                                          └──────────────────┘
```

**Key architecture decisions:**
- **Decouple ingestion from processing** via SQS — the upload API never blocks on processing; queue absorbs burst traffic.
- **Worker Deployment scaled by KEDA** on `queueLength` (messages per pod), not CPU — backlog is the real signal; scale to zero when idle to save cost.
- **Idempotency**: each message processed exactly-once-ish via a dedup key (S3 object key), so retries after a Pod crash mid-processing don't duplicate results.
- **Dead-letter queue** for poison messages (repeatedly failing documents) so they don't block the queue or loop workers forever.
- **Resource requests/limits per worker** sized from load-testing a single document's actual CPU/memory footprint — direct input to how many workers fit per node and HPA/KEDA math.
- **PodDisruptionBudget** on workers so cluster upgrades/node scale-down don't kill too many in-flight jobs at once (or use graceful shutdown + checkpointing if jobs are resumable).
- **Observability**: queue depth + processing latency + failure rate are the primary SLIs; alert on backlog growing faster than drain rate.

## 2. Design Exercise: Multi-Tenant SaaS Platform

```
Namespace-per-tenant (soft multi-tenancy)
├── ResourceQuota (CPU/memory/pod count ceiling per tenant)
├── NetworkPolicy (default-deny + only same-tenant + shared infra allowed)
├── RBAC (tenant admins scoped to their own namespace only)
├── Pod Security Admission: restricted
└── Dedicated ServiceAccount per tenant workload
```

- **Tiering:** free-tier tenants share node pools (bin-packed, tolerant of noisy-neighbor risk mitigated by quotas); enterprise tenants get dedicated node pools (taints + tolerations) or even dedicated clusters for compliance/isolation guarantees.
- **Ingress**: one shared Ingress Controller, host-based routing per tenant (`tenant-a.app.com`), or a wildcard cert + dynamic backend resolution.
- **Data isolation**: separate database schemas/instances per tenant (or row-level security) — Kubernetes-level isolation alone is not sufficient for data compliance.
- **Escalation path for stronger isolation**: vCluster (virtual control plane per tenant) or fully separate clusters for tenants with strict regulatory/compliance requirements.
- **Cost attribution**: label every resource with `tenant=<id>`, feed into a cost-monitoring tool (Kubecost) for chargeback.

## 3. Zero-Downtime Deployment Design

Requires **all** of the following together, not any single piece:
1. **Readiness probes** — new Pods only receive traffic once truly ready.
2. **RollingUpdate strategy** with `maxUnavailable: 0, maxSurge: 1` (or higher) — never drop below desired capacity.
3. **PreStop hook + sleep** — delay SIGTERM slightly so in-flight connections drain from the LB/Service before the container starts shutting down.
4. **Graceful shutdown in-app** — handle SIGTERM, stop accepting new requests, finish in-flight ones, respect `terminationGracePeriodSeconds`.
5. **PodDisruptionBudget** — protects against voluntary disruptions (node drains) compounding with a rollout.
6. **Database migrations** handled as backward-compatible, separate step (expand/contract pattern) — never breaking the currently-running old version mid-rollout.

## 4. Rolling vs Blue/Green vs Canary

| Strategy | How | Rollback speed | Resource cost | Risk exposure |
|----------|-----|-----------------|-----------------|-----------------|
| **Rolling** | Gradually replace old Pods with new ones, in place | Moderate (undo triggers reverse rollout) | Low (brief overlap only) | Partial exposure to all users simultaneously during rollout |
| **Blue/Green** | Deploy full new "green" environment alongside "blue"; switch traffic (Service selector/Ingress) all at once | Instant (flip back to blue) | High (2x full environments running simultaneously) | Zero exposure until cutover, then 100% exposure at once |
| **Canary** | Route a small % of traffic to new version, monitor, gradually increase | Fast (shift weight back to 0%) | Low-moderate (small extra canary replica set) | Minimal — only a small % of users see issues before automatic/manual rollback |

**When to use which:** Rolling = default for most stateless services. Blue/Green = when you need instant, guaranteed rollback and can afford double infrastructure briefly (e.g., major version bumps). Canary = when you want data-driven confidence (error rate/latency metrics) before full rollout, ideal for high-risk changes at scale (often paired with Argo Rollouts + automated analysis).

## 5. Queue-Based Worker Scaling (Recap + Architecture Lens)

- Producers write to a durable queue (SQS/RabbitMQ/Kafka); workers are a Deployment scaled by **KEDA** on queue depth, not CPU.
- Enables **scale-to-zero** during idle periods (major cost lever) and rapid scale-out during bursts (bounded by `maxReplicaCount` and downstream dependency limits — e.g., don't overwhelm a database).
- Backpressure handling: cap concurrency per worker, use a DLQ for poison messages, and monitor "oldest message age" as a leading indicator of SLA breach risk (better than raw queue length alone).

## 6. Multi-AZ High Availability — Full Picture

```
Control plane: 3+ replicas across 3 AZs (managed automatically on EKS)
Worker nodes:  node groups spread across 3 AZs
Workloads:     topologySpreadConstraints (zone) + PodDisruptionBudget
Storage:       EBS is AZ-bound (plan replica placement accordingly);
               EFS/S3 are regional (no AZ constraint)
Data tier:     Multi-AZ RDS / DynamoDB global tables for the same guarantee
                at the data layer — compute HA is worthless if the DB is single-AZ
```

**Interview soundbite:** "HA isn't just `replicas: 3` — it's every layer (control plane, nodes, Pods, storage, and the database) independently surviving the loss of one failure domain, and testing that assumption by actually killing an AZ in staging."

## 7. Resource Isolation & Quotas — Architecture View

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: {name: team-quota, namespace: team-a}
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    pods: "200"
---
apiVersion: v1
kind: LimitRange
metadata: {name: defaults, namespace: team-a}
spec:
  limits:
  - type: Container
    default: {cpu: "500m", memory: "256Mi"}
    defaultRequest: {cpu: "100m", memory: "128Mi"}
```

- **ResourceQuota** = ceiling per namespace (prevents one team from starving the cluster).
- **LimitRange** = sane defaults + min/max per container (prevents individual Pods from being unbounded or absurdly small).
- Combined with **priority classes**, ensures critical workloads preempt non-critical ones under contention rather than first-come-first-served.

## 8. Security Boundaries — Architecture View

```
Identity:   RBAC (humans via OIDC groups) + ServiceAccounts (workloads)
Admission:  Pod Security Admission (restricted) + policy engine (Kyverno/OPA)
Network:    Default-deny NetworkPolicy + explicit allow-lists
Runtime:    Non-root, read-only rootfs, dropped capabilities, seccomp
Secrets:    External secret store (Secrets Manager/Vault) + encryption at rest
Supply chain: Image scanning + signed images + registry allow-list
Audit:      API server audit logs shipped to SIEM
```

**Interview soundbite:** "Security in Kubernetes is defense in depth — no single control is sufficient. RBAC without NetworkPolicy still lets a compromised Pod scan the whole flat network; NetworkPolicy without Pod Security still lets a container run as root and escape. I design all layers together."

## 9. Troubleshoot a Production Latency Incident — Framework

```
1. Scope: Is it all requests or a subset? All Pods or specific ones? All AZs?
2. Check the obvious first: kubectl top pods/nodes -- CPU throttling? Memory pressure?
3. Check HPA: are we under-scaled, replicas maxed out, or thrashing?
4. Check dependencies: DB connection pool exhausted? Downstream service degraded?
5. Check probes: readiness flapping -> pods cycling in/out of Endpoints -> uneven load
6. Check recent changes: new deploy, config change, traffic pattern shift (correlate with timeline)
7. Check node-level: noisy neighbor, disk I/O saturation, network throttling
8. Use tracing (OpenTelemetry) to pinpoint WHICH hop in the request path is slow
9. Mitigate first (scale out, rollback, restart bad pods), root-cause after
```

**Interview soundbite:** "I always separate mitigation from root cause during an incident — scale out or roll back to restore service first, then investigate with logs/traces/metrics without time pressure."

## 10. Mock Staff/Lead Interview Questions — Practice These Out Loud

1. Walk me through what happens end-to-end when you run `kubectl apply -f deployment.yaml`.
2. How would you design zero-downtime deployments for a stateful service with active WebSocket connections?
3. Your cluster autoscaler isn't adding nodes even though Pods are Pending — how do you debug it?
4. A Pod is stuck in CrashLoopBackOff in production at 3am — walk me through your exact steps.
5. How do you decide between Deployment and StatefulSet for a new service?
6. Design a system that processes 10 million documents with a strict cost budget — what auto-scales, and on what signal?
7. How would you isolate two competing teams sharing the same cluster, and what are the limits of that isolation?
8. Explain the tradeoffs between Rolling, Blue/Green, and Canary — when would you pick each?
9. How does a Service actually route traffic to a Pod — trace it from kube-proxy down to iptables/IPVS?
10. What's your incident response process when p99 latency triples in production?
11. Compare ECS and EKS for a company just starting on containers — what would you recommend and why?
12. How do you handle secrets in a way that satisfies a security audit?

## 11. Final Self-Test

> Can I take a blank sheet and design a production Kubernetes platform, explain every major component, identify failure modes, and justify my choices without relying on memorized YAML?

Practice by sketching (on paper/whiteboard, no notes):
1. Full control-plane + data-plane architecture diagram.
2. Request flow: Internet → LB → Ingress → Service → Pod → DB.
3. Deployment rollout lifecycle with all reconciliation loops named.
4. A full incident-response runbook for "service is down."

**Completion date:** ____________________ &nbsp;&nbsp; **Confidence:** ⬜ Beginner ⬜ Working ⬜ Strong ⬜ Interview-ready
