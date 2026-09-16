# Day 8 — Ingress & External Traffic

## Checklist
- [ ] Ingress
- [ ] Ingress Controller
- [ ] Load Balancer
- [ ] Host-based routing
- [ ] Path-based routing
- [ ] TLS termination
- [ ] Configure local ingress
- [ ] Understand AWS ALB + AWS Load Balancer Controller conceptually

---

## 1. Ingress vs Service — Why Ingress Exists

- A `LoadBalancer` Service gives you **one cloud load balancer per Service** — expensive and wasteful if you have 20 microservices each needing external HTTP access.
- **Ingress** is an L7 (HTTP/HTTPS) routing object: a single entry point that fans out to many backend Services based on hostname/path — one load balancer for the entire cluster (or a small number).

```
Internet → Cloud LB → Ingress Controller (pod) → reads Ingress rules → routes to Service → Pod
```

## 2. Ingress Is Just Rules — Ingress Controller Does the Work

- **Ingress** (the object) is purely declarative config — a set of routing rules. On its own it does **nothing**.
- **Ingress Controller** (nginx-ingress, AWS Load Balancer Controller, Traefik, HAProxy) is the actual running component (usually a Deployment/DaemonSet) that watches Ingress objects and configures a real proxy (or cloud LB) to implement those rules.
- You must install an Ingress Controller — Kubernetes doesn't ship with one by default.

## 3. Host-Based and Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["shop.example.com", "api.example.com"]
    secretName: example-tls
  rules:
  # Host-based routing
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: shop-frontend, port: {number: 80}}
  - host: api.example.com
    http:
      paths:
      # Path-based routing within a host
      - path: /v1
        pathType: Prefix
        backend:
          service: {name: api-v1, port: {number: 80}}
      - path: /v2
        pathType: Prefix
        backend:
          service: {name: api-v2, port: {number: 80}}
```

- **Host-based:** different domain names route to entirely different backend Services (multi-tenant, multi-app on one Ingress).
- **Path-based:** same domain, different URL prefixes route to different Services (API versioning, microservice-per-path).

## 4. TLS Termination

- TLS certs are stored as a `kubernetes.io/tls` Secret and referenced in `spec.tls`.
- The Ingress Controller **terminates TLS** — decrypts HTTPS at the edge, then typically forwards plain HTTP to the backend Pod (unless you configure end-to-end TLS/re-encryption).
- In production, certs are usually automated via **cert-manager** + Let's Encrypt (`ClusterIssuer` + annotation `cert-manager.io/cluster-issuer`).

## 5. Hands-On — Local Ingress (kind/minikube)

```bash
# Install nginx ingress controller (kind)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s

kubectl create deployment web1 --image=hashicorp/http-echo -- -text="app one" -listen=:5678
kubectl expose deployment web1 --port=80 --target-port=5678

kubectl create deployment web2 --image=hashicorp/http-echo -- -text="app two" -listen=:5678
kubectl expose deployment web2 --port=80 --target-port=5678

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend: {service: {name: web1, port: {number: 80}}}
      - path: /app2
        pathType: Prefix
        backend: {service: {name: web2, port: {number: 80}}}
EOF

curl localhost/app1
curl localhost/app2
```

## 6. AWS ALB + AWS Load Balancer Controller (Conceptual)

```
Internet → ALB (Application Load Balancer, AWS-managed)
             │  provisioned/configured by
             ▼
   AWS Load Balancer Controller (runs as pods in the EKS cluster,
   watches Ingress objects, calls AWS APIs via IAM role)
             │
             ▼
   Target Group → Pod IPs (if using IP target-type + VPC CNI)
                or Node ports (instance target-type)
```

- On EKS, instead of nginx-ingress running its own in-cluster proxy, the **AWS Load Balancer Controller** watches `Ingress` objects and provisions a **real AWS ALB** via the AWS API (using IAM permissions granted through IRSA/Pod Identity).
- With `target-type: ip`, the ALB routes directly to Pod IPs (via VPC CNI, since Pods get real VPC IPs on EKS) — bypassing kube-proxy/NodePort entirely for lower latency.
- TLS certs are typically managed via **ACM** (AWS Certificate Manager), referenced by annotation rather than a K8s TLS Secret.
- This gives you a fully AWS-native, auto-scaling, highly available load balancer without running your own ingress proxy pods.

## 7. Interview Points
- "Ingress is just a routing spec; the Ingress Controller is the actual proxy/software that implements it — you always need to install one."
- "Ingress lets me consolidate dozens of HTTP services behind one load balancer using host- and path-based rules, instead of provisioning a cloud LB per Service."
- "On EKS, the AWS Load Balancer Controller translates Ingress objects into real ALBs, and with IP target-type it routes straight to Pod IPs via the VPC CNI."
