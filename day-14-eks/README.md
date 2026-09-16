# Day 14 — AWS EKS

## Checklist
- [ ] EKS architecture
- [ ] Managed control plane
- [ ] Managed Node Groups
- [ ] EKS networking
- [ ] AWS VPC CNI concept
- [ ] ECR
- [ ] ALB / AWS Load Balancer Controller
- [ ] IAM and Pod Identity / IRSA concepts
- [ ] EBS / EFS
- [ ] CloudWatch integration concepts
- [ ] ECS vs EKS tradeoffs
- [ ] Optionally deploy a small workload to EKS

---

## 1. EKS Architecture Overview

```
┌───────────────────────── AWS-MANAGED ─────────────────────────┐
│  Control Plane (API Server, etcd, Scheduler, Ctrl Manager)      │
│  — runs across multiple AZs, fully managed, you never see it   │
│  — you only interact via the EKS API endpoint                   │
└─────────────────────────────────────────────────────────────────┘
                              │
┌───────────────────────── YOU MANAGE ──────────────────────────┐
│  Data Plane: EC2 worker nodes (Managed Node Groups / self-      │
│  managed / Fargate) running kubelet, kube-proxy, VPC CNI,      │
│  and your Pods                                                  │
└─────────────────────────────────────────────────────────────────┘
```

- EKS = AWS runs and guarantees the **control plane** (HA across 3 AZs, patched, backed up automatically); you are responsible for the **data plane** (worker nodes) unless using **Fargate** (fully serverless Pods, no nodes to manage at all).

## 2. Managed Control Plane

- You don't SSH into or see API Server/etcd instances — AWS manages their availability, scaling, patching, and etcd backups.
- You interact only through the EKS-provided API endpoint (can be public, private, or both).
- You pay a flat hourly fee per cluster for this managed control plane, separate from EC2/Fargate compute costs.

## 3. Managed Node Groups

- AWS-managed **EC2 Auto Scaling Groups** wired into EKS — handles node provisioning, joining the cluster, and rolling AMI/version upgrades for you.
- Alternative options: **self-managed node groups** (you control the ASG/launch template fully) or **Fargate profiles** (no EC2 nodes at all — AWS runs each Pod in its own micro-VM).

| Option | You manage | Best for |
|--------|-----------|----------|
| Managed Node Groups | Just the ASG config (instance type, scaling) | Most general workloads — good balance of control and simplicity |
| Self-managed nodes | AMI, launch template, lifecycle scripts | Custom AMIs, specialized bootstrapping needs |
| Fargate | Nothing — pay per Pod vCPU/memory | Bursty/unpredictable workloads, no interest in node ops, strict per-Pod isolation |

## 4. EKS Networking — AWS VPC CNI

- Unlike overlay-network CNIs (Flannel/Calico VXLAN), the **AWS VPC CNI** assigns each Pod a **real IP address from the VPC's subnet** (via ENIs/secondary IPs attached to the worker node's ENI).
- Benefit: Pods are natively routable within the VPC — no encapsulation overhead, and ALBs/NLBs can target Pod IPs directly (`target-type: ip`).
- Constraint: **IP exhaustion** is a real concern — number of Pods per node is limited by ENI/IP capacity of the instance type; large clusters need careful subnet sizing (or prefix delegation to increase density).

## 5. ECR (Elastic Container Registry)

- AWS's private container image registry, integrated with IAM (no separate registry credentials needed — node IAM role grants pull access).
- Supports image scanning (vulnerability detection), lifecycle policies (auto-expire old images), and replication across regions.

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
docker push <account>.dkr.ecr.<region>.amazonaws.com/myapp:v1
```

## 6. ALB + AWS Load Balancer Controller

- The **AWS Load Balancer Controller** runs as pods inside the cluster, watches `Ingress` (→ provisions ALBs) and `Service type=LoadBalancer` (→ provisions NLBs).
- With `target-type: ip`, traffic goes straight from the ALB to Pod IPs (via VPC CNI), skipping kube-proxy/NodePort — lower latency, more accurate health checks.
- TLS is typically handled by referencing an **ACM** certificate ARN via annotation, rather than a K8s TLS Secret.

## 7. IAM and Pod Identity / IRSA

**Problem:** Pods need AWS permissions (e.g., read from S3, write to DynamoDB) without hardcoding AWS access keys.

- **IRSA (IAM Roles for Service Accounts)** — the older, still widely used approach: an OIDC identity provider is associated with the cluster; a Kubernetes ServiceAccount is annotated with an IAM role ARN; AWS SDKs inside the Pod automatically assume that role via a projected, short-lived web identity token.
- **EKS Pod Identity** — the newer, simpler mechanism: associates an IAM role directly with a ServiceAccount via the EKS API (no OIDC provider setup, no trust policy juggling), and rotates credentials automatically via an in-cluster agent.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/app-s3-access   # IRSA style
```

- Both approaches mean: **no static AWS credentials in Secrets or environment variables**, and permissions scoped per-application (per-ServiceAccount) rather than per-node.

## 8. EBS / EFS on EKS

| | EBS CSI Driver | EFS CSI Driver |
|---|----------------|-----------------|
| Storage type | Block storage | Network file system (NFS) |
| Access mode | `ReadWriteOnce` only — one node at a time | `ReadWriteMany` — many pods/nodes concurrently |
| AZ-bound? | Yes — an EBS volume lives in one AZ, so Pods using it must be scheduled in that same AZ | No — EFS is regional, accessible from any AZ |
| Typical use | Databases, single-writer workloads (StatefulSets) | Shared config/media, multi-writer workloads, CMS uploads |

## 9. CloudWatch Integration

- **CloudWatch Container Insights** collects cluster/node/pod-level metrics and logs without deploying your own Prometheus/Grafana stack (tradeoff: less flexible querying, AWS-native cost model).
- **Fluent Bit** (as a DaemonSet) is the standard way to ship container logs to CloudWatch Logs on EKS.
- Many teams still layer **Prometheus + Grafana** on top or instead, for more powerful querying (PromQL) and portability across clouds.

## 10. ECS vs EKS — Tradeoffs

| | ECS | EKS |
|---|-----|-----|
| Orchestration model | AWS-proprietary, simpler task/service model | Kubernetes — open standard, portable |
| Learning curve | Lower — fewer concepts | Higher — full K8s object model |
| Portability | AWS-only | Runs anywhere K8s runs (multi-cloud, on-prem, hybrid) |
| Ecosystem | Smaller, AWS-native tools only | Massive CNCF ecosystem (Helm, ArgoCD, service mesh, operators...) |
| Control plane cost | Free (control plane not separately billed) | Flat hourly fee per cluster |
| Talent availability | Smaller pool, AWS-specific skill | Much larger pool, transferable skill across employers/clouds |
| Best for | Teams fully committed to AWS, wanting simplicity | Teams wanting portability, complex orchestration needs, multi-cloud strategy, or already invested in K8s tooling |

**Interview framing:** "ECS is simpler and cheaper for AWS-only shops with straightforward workloads. EKS makes sense when you need Kubernetes' richer scheduling/extensibility model, want cloud portability, or your org already has Kubernetes expertise and tooling investment — the tradeoff is more operational complexity and a control plane cost."

## 11. Optional Hands-On (requires AWS account)

```bash
eksctl create cluster --name mastery-cluster --region us-east-1 \
  --nodegroup-name workers --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 4

kubectl get nodes
kubectl create deployment hello --image=nginx:1.27-alpine
kubectl expose deployment hello --port=80 --type=LoadBalancer
kubectl get svc hello   # EXTERNAL-IP will be an AWS-provisioned ELB hostname

eksctl delete cluster --name mastery-cluster --region us-east-1   # avoid ongoing charges
```

## 12. Interview Points
- "EKS gives you a fully AWS-managed, multi-AZ control plane — I only operate the data plane, which can be managed node groups, self-managed EC2, or Fargate for a fully serverless Pod experience."
- "The AWS VPC CNI gives Pods real VPC IPs, which is what allows ALBs to target Pods directly — but it means IP address planning matters a lot more than with an overlay CNI."
- "For AWS credentials in Pods, I use IRSA or EKS Pod Identity — never static keys — so permissions are scoped per-ServiceAccount and rotate automatically."
- "I'd choose ECS for AWS-only simplicity, EKS when I need portability, a broader ecosystem, or the org already has Kubernetes skills to leverage."
