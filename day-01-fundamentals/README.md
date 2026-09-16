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
5. **Scheduler** notices a Pod with no `spec.nodeName` set, filters and scores eligible nodes, then binds the Pod to a chosen node. The scheduler does not write to etcd directly: it sends the binding through the API Server, which persists the updated object to etcd.
6. The **kubelet** on the selected node watches the API Server for Pods assigned to its `spec.nodeName`. It sees the assignment and instructs the **container runtime** (containerd) to pull the image and start the container(s).
7. The kubelet continuously reports Pod and container status to the API Server. The API Server persists that status in etcd. Kubelets and controllers do not normally access etcd directly.
8. Controllers (for example, the Deployment and ReplicaSet controllers) keep watching and reconciling so that actual state matches desired state indefinitely.

**Key point for interviews:** `kubectl apply` never talks to a node directly — everything flows through the API Server, and the actual placement/execution happens asynchronously via the watch/reconcile pattern.

### What does “assign the Pod to a node” mean?

Initially, a newly created Pod has no `spec.nodeName`, so it is **Pending** and has not been assigned to a node. The scheduler evaluates the Pod's resource requests, node selectors, affinity rules, taints and tolerations, topology constraints, and other scheduling rules. It chooses a node and records that choice as `spec.nodeName` by calling the API Server. This is called **binding**.

The selected kubelet maintains a watch on the API Server. The API Server sends it the relevant Pod event, or the kubelet observes the change when its watch is re-established. The kubelet then creates the Pod's sandbox and asks the container runtime to pull images and start containers. The kubelet does not learn the assignment by reading etcd directly.

### What is a PodSpec?

A **PodSpec** is the `spec` section of a Pod object: the desired description of how the Pod should run. It includes fields such as:

- container names and images
- ports, environment variables, volumes, and probes
- resource requests and limits
- node selectors, affinity, tolerations, and restart policy

For example, in this manifest, the `containers` list and `restartPolicy` are part of the PodSpec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27-alpine
  restartPolicy: Always
```

The kubelet uses the PodSpec as its instruction, while the Pod's `status` records what is actually happening. Kubernetes compares the two rather than treating a successful API request as proof that a container is already running.

### What is the watch/reconcile pattern?

- **Watch:** components subscribe to the API Server for changes to the objects they care about. For example, the scheduler watches unscheduled Pods, and a kubelet watches Pods assigned to its node.
- **Reconcile:** a component compares the desired state in the API objects with the observed state, then takes an action to reduce the difference.

This is asynchronous and continuous. A controller may create a replacement Pod, a scheduler may bind it, and a kubelet may restart a failed container. Each action produces another API event, and the loops continue until the desired and actual states converge. Components watch the API Server; the API Server is the boundary through which state is read and changed, with etcd providing persistence behind it.

### What happens if a node goes down?

1. The kubelet normally sends heartbeats through the API Server using a **Node Lease** and Node status updates. If those stop, the control plane eventually marks the node `NotReady`.
2. The Node controller detects the failure and, subject to configured timing and workload rules, marks Pods on that node as failed or evicts them.
3. If those Pods belong to a Deployment, ReplicaSet, or StatefulSet, its controller notices the missing replicas and creates replacement Pods. The scheduler can place replacements on healthy nodes.
4. A standalone Pod created directly with `kubectl run` is not recreated by a Deployment or ReplicaSet controller. It may remain associated with the failed node until the node or Pod is handled, so production workloads are normally managed by a controller.

The exact timing depends on node-monitor and eviction settings, and a temporary network partition can look like a node failure. Kubernetes cannot immediately know whether a silent node is destroyed or merely disconnected.

### Is `kubectl run nginx --image=nginx` the same?

The request still follows the same control-plane path: `kubectl` calls the API Server, authentication/authorization and admission run, the object is persisted, and then the scheduler and kubelet act asynchronously. The difference is how the desired object is created:

| Command | Meaning |
|---------|---------|
| `kubectl apply -f pod.yaml` | Declarative: submit a complete manifest and repeatedly apply changes to that declared object. |
| `kubectl run nginx --image=nginx` | Imperative: ask `kubectl` to construct and create a Pod from command-line arguments. |

Both create a Pod that can be scheduled and run. `kubectl run` does not by itself create a Deployment or provide replica management, rolling updates, or replacement replicas. For a long-lived application, use a Deployment manifest or `kubectl create deployment` instead.
