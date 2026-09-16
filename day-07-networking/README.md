# Day 7 — Kubernetes Networking

## Checklist
- [ ] Pod networking and Pod IPs
- [ ] Why Pod IPs are ephemeral
- [ ] Service
- [ ] ClusterIP
- [ ] NodePort
- [ ] LoadBalancer
- [ ] Kubernetes DNS
- [ ] kube-proxy and CNI at a practical level
- [ ] Deploy frontend + backend and communicate through a Service

---

## 1. Pod Networking Model

Kubernetes networking has three foundational rules (implemented by the CNI plugin):
1. Every Pod gets its **own unique IP** (no NAT between pods).
2. Any Pod can talk to any other Pod's IP directly, on any node, without NAT.
3. Agents on a node (kubelet) can talk to all Pods on that node.

This is why Kubernetes networking feels like "one big flat network" even though Pods are spread across many physical nodes — the CNI plugin (Calico, Cilium, Flannel, VPC CNI...) implements this via overlays (VXLAN) or native routing (BGP) or cloud-native ENIs.

## 2. Why Pod IPs Are Ephemeral

- A Pod's IP is assigned when it's scheduled and **released when the Pod is deleted** — a replacement Pod (even with the same name pattern from a Deployment) gets a **new IP**.
- Pods die and get replaced constantly (crashes, rollouts, node failures, scaling) — hardcoding a Pod IP anywhere is guaranteed to break.
- **This is exactly why Services exist**: they provide a stable virtual IP + DNS name that doesn't change, while transparently load-balancing to whichever Pod IPs are currently healthy.

## 3. Service — The Stable Abstraction

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend            # Selects pods by label -> populates Endpoints
  ports:
  - port: 80                # Service's own port
    targetPort: 8080         # Container port to forward to
```

- A Service continuously watches for Pods matching its `selector` and maintains an **Endpoints/EndpointSlice** object listing their current IPs.
- `kube-proxy` on every node watches Services/Endpoints and programs local rules (iptables or IPVS) so that traffic to the Service's ClusterIP gets load-balanced to one of the healthy backing Pod IPs.

## 4. Service Types

| Type | Exposes | Use case |
|------|---------|----------|
| **ClusterIP** (default) | Internal-only virtual IP, reachable inside the cluster | Service-to-service traffic (e.g., frontend → backend) |
| **NodePort** | Opens a static port (30000-32767) on **every node's** IP, forwards to ClusterIP | Simple external access without a cloud LB; dev/test |
| **LoadBalancer** | Provisions a cloud load balancer (ELB/ALB/Azure LB) pointing at NodePort → ClusterIP | Production external access on cloud providers |
| **ExternalName** | DNS CNAME to an external hostname, no proxying | Referencing external services (e.g., a managed DB) by internal name |

```
LoadBalancer  →  NodePort  →  ClusterIP  →  Pod IPs
(cloud LB)      (node:port)   (virtual IP)  (actual endpoints)
```

Each type is a **superset** of the one below it — a LoadBalancer Service still has a ClusterIP and (usually) a NodePort under the hood.

## 5. Kubernetes DNS

- **CoreDNS** runs as a Deployment in `kube-system`, exposed via its own Service — it's the cluster's internal DNS server.
- Every Service automatically gets a DNS name:
  ```
  <service-name>.<namespace>.svc.cluster.local
  backend.default.svc.cluster.local
  ```
- Pods within the same namespace can just use the short name: `backend`. Cross-namespace: `backend.other-namespace`.
- Pod `/etc/resolv.conf` is automatically configured to query CoreDNS and search these suffixes.

```bash
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --command -- sleep 3600
kubectl exec -it dnsutils -- nslookup backend.default.svc.cluster.local
```

## 6. kube-proxy and CNI — Practical Roles

| Component | Layer | Responsibility |
|-----------|-------|-----------------|
| **CNI plugin** (Calico, Cilium, Flannel, AWS VPC CNI) | Pod-to-Pod networking | Assigns Pod IPs, sets up routes/tunnels so any Pod can reach any Pod cluster-wide |
| **kube-proxy** | Service virtual IP → Pod IP translation | Watches Services/Endpoints, programs iptables/IPVS rules on every node so traffic to a ClusterIP gets DNAT'd to a real, healthy Pod IP |

They solve **different problems**: CNI = "can packets get from Pod A to Pod B at all"; kube-proxy = "how does a stable Service IP become one of many changing Pod IPs."

## 7. Hands-On Lab — Frontend + Backend via Service

```yaml
# file: frontend-backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector: {matchLabels: {app: backend}}
  template:
    metadata: {labels: {app: backend}}
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo
        args: ["-text=hello from backend", "-listen=:8080"]
        ports: [{containerPort: 8080}]
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector: {app: backend}
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector: {matchLabels: {app: frontend}}
  template:
    metadata: {labels: {app: frontend}}
    spec:
      containers:
      - name: frontend
        image: curlimages/curl
        command: ["sh", "-c", "while true; do curl -s http://backend; sleep 5; done"]
```

```bash
kubectl apply -f frontend-backend.yaml
kubectl logs -f deployment/frontend
# Should print "hello from backend" every 5s, load-balanced across 2 backend pods

kubectl get endpoints backend
kubectl get svc backend

# Expose externally for testing (kind/minikube)
kubectl expose deployment backend --name=backend-nodeport --type=NodePort --port=80 --target-port=8080
kubectl get svc backend-nodeport
```

## 8. Interview Points
- "Pod IPs are ephemeral by design — Services give you a stable virtual IP and DNS name that survives Pod churn."
- "kube-proxy doesn't do L7 routing — it's L4 (TCP/UDP) NAT via iptables/IPVS, driven by whatever the CNI made possible at the Pod networking layer."
- "ClusterIP, NodePort, and LoadBalancer are layered — a LoadBalancer Service is still backed by a ClusterIP internally."
