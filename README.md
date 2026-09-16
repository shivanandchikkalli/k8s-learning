# 15-Day Kubernetes Mastery Tracker

**Goal:** Move from "I have used EKS" to confidently designing, deploying, troubleshooting, scaling, securing, and explaining production Kubernetes systems.

**Recommended setup:** Days 1–13 local Kubernetes (kind/minikube); Day 14 EKS concepts/optional AWS practice; Day 15 Staff-level architecture and mock interview.

---

## Progress Tracker

| Day | Topic | Link | Status |
|-----|-------|------|--------|
| 1 | Kubernetes Fundamentals | [day-01-fundamentals](./day-01-fundamentals/README.md) | ⬜ |
| 2 | Pods, ReplicaSets & Deployments | [day-02-pods-deployments](./day-02-pods-deployments/README.md) | ⬜ |
| 3 | API & Controllers | [day-03-api-controllers](./day-03-api-controllers/README.md) | ⬜ |
| 4 | ConfigMaps & Secrets | [day-04-configmaps-secrets](./day-04-configmaps-secrets/README.md) | ⬜ |
| 5 | Probes & Application Lifecycle | [day-05-probes-lifecycle](./day-05-probes-lifecycle/README.md) | ⬜ |
| 6 | Resource Management | [day-06-resource-management](./day-06-resource-management/README.md) | ⬜ |
| 7 | Kubernetes Networking | [day-07-networking](./day-07-networking/README.md) | ⬜ |
| 8 | Ingress & External Traffic | [day-08-ingress](./day-08-ingress/README.md) | ⬜ |
| 9 | Storage & Stateful Workloads | [day-09-storage-stateful](./day-09-storage-stateful/README.md) | ⬜ |
| 10 | Scaling | [day-10-scaling](./day-10-scaling/README.md) | ⬜ |
| 11 | Scheduling & High Availability | [day-11-scheduling-ha](./day-11-scheduling-ha/README.md) | ⬜ |
| 12 | Kubernetes Security | [day-12-security](./day-12-security/README.md) | ⬜ |
| 13 | Production Troubleshooting | [day-13-troubleshooting](./day-13-troubleshooting/README.md) | ⬜ |
| 14 | AWS EKS | [day-14-eks](./day-14-eks/README.md) | ⬜ |
| 15 | Staff-Level Architecture & Interview | [day-15-architecture-interview](./day-15-architecture-interview/README.md) | ⬜ |
| + | Helm Deep Dive (companion, staff-level) | [helm-deep-dive](./helm-deep-dive/README.md) | ⬜ |
| + | Terraform Deep Dive (companion, staff-level) | [terraform-deep-dive](./terraform-deep-dive/README.md) | ⬜ |

Mark ⬜ → ✅ as you complete each day. Each day folder has its own checklist, hands-on labs, and interview questions.

Helm basics are introduced inline on Days 9, 12, and 14 (installing charts as you touch storage, security tooling, and EKS). The [helm-deep-dive](./helm-deep-dive/README.md) track covers chart authoring, templating, hooks, and testing for staff-level depth — work through it alongside Days 9–14.

Terraform basics are introduced inline on Day 14 (provisioning the EKS cluster, VPC, and IAM/IRSA trust as code). The [terraform-deep-dive](./terraform-deep-dive/README.md) track covers state management, modules, workspaces, drift, and the Terraform/Kubernetes boundary for staff-level depth — work through it alongside Day 14.

---

## Setup

```bash
# Option 1: kind (multi-node, recommended)
kind create cluster --config kind-config.yaml

# Option 2: minikube
minikube start --cpus=4 --memory=8192

# Verify
kubectl cluster-info
kubectl get nodes
```

---

## Day-15 Final Kubernetes Mastery Checklist

Use this as the final gate. Check an item only if you can explain it clearly and, where applicable, demonstrate it hands-on.

### 1. Core Architecture
- [ ] I can draw the Kubernetes architecture from memory.
- [ ] I can explain API Server, etcd, Scheduler, Controller Manager and Kubelet.
- [ ] I understand desired state and reconciliation.
- [ ] I can explain what happens after `kubectl apply`.

### 2. Workloads
- [ ] I can explain Pod vs ReplicaSet vs Deployment.
- [ ] I can perform rolling updates and rollbacks.
- [ ] I understand StatefulSet and when it is needed.
- [ ] I can explain what happens when a Pod or node fails.

### 3. Networking
- [ ] I can explain Pod IPs and why Services are needed.
- [ ] I understand ClusterIP, NodePort and LoadBalancer.
- [ ] I can explain Kubernetes DNS.
- [ ] I can explain Ingress and an Ingress Controller.
- [ ] I can trace a request from Internet → Load Balancer → Ingress → Service → Pod.

### 4. Configuration & Storage
- [ ] I can use ConfigMaps and Secrets appropriately.
- [ ] I understand external secret-management patterns.
- [ ] I can explain PV, PVC and StorageClass.
- [ ] I can explain stateful vs stateless workloads.

### 5. Reliability & Scaling
- [ ] I understand readiness, liveness and startup probes.
- [ ] I understand graceful shutdown.
- [ ] I can explain requests vs limits and OOMKilled.
- [ ] I can explain HPA vs Cluster Autoscaler.
- [ ] I can design multi-AZ replica distribution.
- [ ] I understand PodDisruptionBudget at a practical level.

### 6. Scheduling
- [ ] I understand node affinity and node selectors.
- [ ] I understand taints and tolerations.
- [ ] I understand Pod anti-affinity and topology spread constraints.
- [ ] I can explain how resource requests affect scheduling.

### 7. Security
- [ ] I understand RBAC.
- [ ] I understand Role/ClusterRole and bindings.
- [ ] I understand ServiceAccounts.
- [ ] I can explain SecurityContext and non-root containers.
- [ ] I understand NetworkPolicy.
- [ ] I can describe how I would isolate workloads/tenants.

### 8. Troubleshooting
- [ ] I can troubleshoot CrashLoopBackOff.
- [ ] I can troubleshoot Pending Pods.
- [ ] I can troubleshoot ImagePullBackOff.
- [ ] I can troubleshoot OOMKilled.
- [ ] I can troubleshoot Service connectivity.
- [ ] I can troubleshoot Ingress/external connectivity.
- [ ] I know the key kubectl commands without looking them up.

### 9. AWS EKS
- [ ] I can explain EKS architecture.
- [ ] I understand managed control plane vs worker nodes.
- [ ] I understand EKS networking at a high level.
- [ ] I understand ECR, ALB and AWS Load Balancer Controller.
- [ ] I understand IAM/Pod Identity concepts.
- [ ] I can compare ECS and EKS using concrete tradeoffs.

### 10. Staff-Level Architecture
- [ ] I can design a 10-million-document processing platform.
- [ ] I can design queue-based worker scaling.
- [ ] I can design a multi-tenant Kubernetes platform.
- [ ] I can design zero-downtime deployments.
- [ ] I can explain Rolling vs Blue/Green vs Canary.
- [ ] I can reason about HA, failure domains, security, scaling and observability.
- [ ] I can troubleshoot a production incident systematically.
- [ ] I can defend Kubernetes architecture decisions and tradeoffs in an interview.

**Final self-test:** Can I take a blank sheet and design a production Kubernetes platform, explain every major component, identify failure modes, and justify my choices without relying on memorized YAML?

**Completion date:** ____________________ &nbsp;&nbsp; **Confidence:** ⬜ Beginner ⬜ Working ⬜ Strong ⬜ Interview-ready
