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

## 12. Provisioning EKS with Terraform (Instead of eksctl)

`eksctl` is fine for quick labs, but production clusters are provisioned as code
so the cluster, VPC, IAM roles, and node groups are versioned, reviewable, and
reproducible. The standard approach uses the official
`terraform-aws-modules/eks/aws` module rather than hand-rolling every resource.

```hcl
# file: main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "mastery-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true   # cost-saving for non-prod; use one NAT per AZ for prod HA
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "mastery-cluster"
  cluster_version = "1.31"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size       = 2
      max_size       = 4
      desired_size   = 2
    }
  }
}

# IAM role for the AWS Load Balancer Controller, trusted via the cluster's OIDC provider
module "lb_controller_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name                              = "aws-load-balancer-controller"
  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}
```

```bash
terraform init
terraform plan -out=tfplan     # ALWAYS review the plan before applying
terraform apply tfplan

aws eks update-kubeconfig --name mastery-cluster --region us-east-1
kubectl get nodes

terraform destroy              # tear down VPC + cluster + node groups together
```

### Why This Matters at Staff Level
- **One `terraform plan`/`apply` provisions the VPC, cluster, node groups, and the
  IAM role/OIDC trust for IRSA together** — no manual clicking in the AWS console,
  no drift between environments.
- The **module boundary matches the Terraform/Helm boundary from Section 13**:
  Terraform owns the IAM role and its trust policy (`lb_controller_irsa`); Helm
  only references the resulting ServiceAccount name — neither tool tries to own
  the other's responsibility.
- State (`terraform.tfstate`) should live in a **remote backend** (S3 + DynamoDB
  lock table) for any shared/production cluster — local state is only acceptable
  for solo learning labs like this one.

```hcl
# file: backend.tf — remote state for anything beyond a personal lab
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "clusters/mastery-cluster/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

For a full staff-level treatment of Terraform (state management, modules, workspaces,
drift detection, import), see [terraform-deep-dive/README.md](../terraform-deep-dive/README.md).

## 13. Installing the AWS Load Balancer Controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=mastery-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

- `serviceAccount.create=false` because in production you pre-create the
  ServiceAccount via Terraform/IRSA with the correct IAM role annotation —
  Helm should not own IAM-linked identity.
- Standard pattern: **Terraform provisions the IAM role + OIDC trust, Helm installs
  the controller referencing that pre-existing ServiceAccount.**

## 14. Interview Points
- "EKS gives you a fully AWS-managed, multi-AZ control plane — I only operate the data plane, which can be managed node groups, self-managed EC2, or Fargate for a fully serverless Pod experience."
- "The AWS VPC CNI gives Pods real VPC IPs, which is what allows ALBs to target Pods directly — but it means IP address planning matters a lot more than with an overlay CNI."
- "For AWS credentials in Pods, I use IRSA or EKS Pod Identity — never static keys — so permissions are scoped per-ServiceAccount and rotate automatically."
- "I'd choose ECS for AWS-only simplicity, EKS when I need portability, a broader ecosystem, or the org already has Kubernetes skills to leverage."
- "I install vendor components like the AWS Load Balancer Controller via Helm, but let Terraform own the IAM role/ServiceAccount binding — infra identity shouldn't be owned by the app-layer tool."
- "Terraform provisions the cluster, VPC, and IAM/OIDC trust as one reviewable plan — I never hand-create infrastructure that Terraform is supposed to own, or state drifts silently."
