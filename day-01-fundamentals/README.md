# Day 1 — Kubernetes Fundamentals

## Checklist
- [ ] Understand containers vs VMs and why Kubernetes exists
- [ ] Cluster, node, pod, container, namespace, kubectl
- [ ] Control plane vs worker nodes
- [ ] API Server, etcd, Scheduler, Controller Manager, Kubelet
- [ ] Install local Kubernetes and run basic kubectl commands
- [ ] Create and inspect a Pod
- [ ] Explain desired state and declarative configuration
- [ ] Interview: What happens when `kubectl apply` is executed?

---

## 1. Containers vs VMs

```
VM: Hardware → Hypervisor → Guest OS (full) → App
Container: Hardware → Host OS → Container Runtime → App (shares kernel)
```

- VMs virtualize hardware; containers virtualize the OS.
- Containers start in seconds, are MBs not GBs, and pack far more densely per host.
- Kubernetes exists because running hundreds/thousands of containers across many machines by hand (placement, restarts, networking, scaling) is not humanly manageable — you need a system that continuously reconciles "what you want" with "what is running."

## 2. Core Vocabulary

| Term | Definition |
|------|-----------|
| **Cluster** | A set of machines (nodes) running Kubernetes, managed as one unit |
| **Node** | A single machine (VM or bare metal) in the cluster |
| **Pod** | Smallest deployable unit — one or more containers sharing network/storage |
| **Container** | The actual running process, packaged with its dependencies |
| **Namespace** | Logical partition within a cluster for isolation/organization |
| **kubectl** | CLI tool that talks to the API Server to manage cluster objects |

## 3. Control Plane vs Worker Nodes

```
┌────────────────────────── CONTROL PLANE ──────────────────────────┐
│  API Server   │   etcd   │   Scheduler   │  Controller Manager    │
│  (front door) │ (state)  │  (placement)  │  (reconciliation)      │
└─────────────────────────────────────────────────────────────────┘
                              │
                 manages/watches
                              ▼
┌────────────────────────── WORKER NODE ────────────────────────────┐
│   kubelet   │   kube-proxy   │   container runtime (containerd)   │
│                        Pods running here                          │
└─────────────────────────────────────────────────────────────────┘
```

- **Control plane** makes global decisions (scheduling, scaling, reacting to failures) and stores cluster state. It does not run your application containers.
- **Worker nodes** run the actual application Pods.

## 4. Control Plane Components

| Component | Role |
|-----------|------|
| **API Server** | The only entry point to the cluster. All reads/writes (kubectl, controllers, kubelets) go through it via REST/HTTPS. Validates and persists to etcd. |
| **etcd** | Distributed, consistent key-value store. The single source of truth for all cluster state. If etcd is lost, cluster state is lost. |
| **Scheduler** | Watches for Pods with no assigned node, picks the best node based on resources, affinity, taints, etc. |
| **Controller Manager** | Runs many control loops (Deployment controller, Node controller, ReplicaSet controller...) that watch current state and drive it toward desired state. |
| **Kubelet** (node) | Agent on every node; ensures containers described in PodSpecs are actually running and healthy. |

## 5. Declarative Configuration & Desired State

- **Imperative**: "Run this container now" (`kubectl run`, `docker run`) — you tell the system the steps.
- **Declarative**: "I want 3 replicas of this image running, always" (`kubectl apply -f deployment.yaml`) — you describe the end state, and Kubernetes continuously works to achieve/maintain it.
- This is the **reconciliation loop**: `desired state (etcd)` vs `actual state (observed from cluster)` → controllers act to close the gap, forever, in a loop.

## 6. Hands-On Lab

```bash
# Set up local cluster
kind create cluster --config ../kind-config.yaml --name mastery
kubectl cluster-info
kubectl get nodes -o wide

# Explore control plane pods (kind runs them as pods in kube-system)
kubectl get pods -n kube-system

# Create a namespace
kubectl create namespace learning

# Create and inspect a pod (imperative, for learning only)
kubectl run nginx --image=nginx:1.27-alpine -n learning
kubectl get pods -n learning
kubectl describe pod nginx -n learning
kubectl get pod nginx -n learning -o yaml

# Declarative version
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-declarative
  namespace: learning
spec:
  containers:
  - name: nginx
    image: nginx:1.27-alpine
EOF

kubectl get pod nginx-declarative -n learning -o wide
kubectl delete namespace learning
```

## 7. Interview: What happens when `kubectl apply` is executed?

1. `kubectl` reads the YAML, converts to JSON, sends an HTTPS request to the **API Server**.
2. API Server **authenticates** the caller (who are you?) and **authorizes** (RBAC — are you allowed?).
3. Request passes through **admission controllers** (mutating, then validating) — may set defaults or reject invalid specs.
4. API Server **persists** the object to **etcd** as the new desired state.
5. **Scheduler** notices a Pod with no `nodeName` set, scores eligible nodes, binds the Pod to a chosen node (writes back to API Server/etcd).
6. **kubelet** on that node is watching the API Server, sees a new Pod assigned to it, and instructs the **container runtime** (containerd) to pull the image and start the container(s).
7. kubelet continuously reports Pod status back to the API Server, which updates etcd.
8. Controllers (e.g., Deployment/ReplicaSet controller) keep watching to ensure actual state matches desired state indefinitely.

**Key point for interviews:** `kubectl apply` never talks to a node directly — everything flows through the API Server, and the actual placement/execution happens asynchronously via the watch/reconcile pattern.
