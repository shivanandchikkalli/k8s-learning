# Day 12 — Kubernetes Security

## Checklist
- [ ] RBAC
- [ ] Role vs ClusterRole
- [ ] RoleBinding vs ClusterRoleBinding
- [ ] ServiceAccount
- [ ] SecurityContext
- [ ] runAsNonRoot
- [ ] Linux capabilities / privileged containers
- [ ] NetworkPolicy
- [ ] Tenant/application isolation concepts
- [ ] Create and test a basic RBAC policy

---

## 1. RBAC — Role-Based Access Control

RBAC answers: "**Who** can do **what** to **which resources**?" It's the **authorization** layer, checked after authentication on every API Server request.

```
Role / ClusterRole   = WHAT actions are allowed (verbs on resources)
RoleBinding / ClusterRoleBinding = WHO gets those permissions (subjects)
```

## 2. Role vs ClusterRole

| | Role | ClusterRole |
|---|------|-------------|
| Scope | Single namespace | Cluster-wide (or reusable across namespaces) |
| Can grant access to | Namespaced resources only (pods, deployments, secrets...) | Namespaced resources AND cluster-scoped resources (nodes, PVs, namespaces themselves) |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev-team
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

## 3. RoleBinding vs ClusterRoleBinding

| | RoleBinding | ClusterRoleBinding |
|---|------------|---------------------|
| Grants permission | Within **one namespace** only | Across the **entire cluster** |
| Can reference | A Role (same ns) OR a ClusterRole (permissions applied only within this ns) | Only a ClusterRole (applied cluster-wide) |

```yaml
# Common reusable pattern: ClusterRole + RoleBinding = 
# cluster-defined permission set, but granted only in one namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-pods
  namespace: dev-team
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: dev-team
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-wide-node-readers
subjects:
- kind: Group
  name: platform-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## 4. ServiceAccount — Identity for Pods

- Every Pod runs **as** a ServiceAccount (default: `default` SA in its namespace, unless specified).
- ServiceAccounts are how **Pods/apps** authenticate to the API Server (as opposed to human users, who typically use certs/OIDC).
- Best practice: create a **dedicated, minimally-privileged ServiceAccount per application**, never rely on the `default` SA with broad permissions.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  namespace: production
automountServiceAccountToken: false   # disable unless the app truly needs to call the API
---
apiVersion: v1
kind: Pod
metadata: {name: api}
spec:
  serviceAccountName: api-sa
  containers: [{name: api, image: myapp:v1}]
```

## 5. SecurityContext — Runtime Hardening

```yaml
spec:
  securityContext:                 # Pod-level
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: app
    image: myapp:v1
    securityContext:               # Container-level (overrides pod-level)
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop: ["ALL"]              # remove all Linux capabilities by default
        add: ["NET_BIND_SERVICE"]  # add back only what's truly needed
```

### `runAsNonRoot`
- Forces the container to run as a non-root UID. If the image's default user is root and no `runAsUser` is set, the Pod **fails to start** rather than silently running as root.
- Reduces blast radius of a container breakout — a compromised non-root process has far fewer privileges on the host if it escapes.

### Linux Capabilities / Privileged Containers
- A **privileged** container (`privileged: true`) has essentially full access to the host (all devices, kernel capabilities, bypasses most isolation) — avoid entirely in production except for specific infra pods (CNI, storage drivers) that genuinely need it.
- **Capabilities** are a finer-grained alternative to all-or-nothing root — e.g., `NET_BIND_SERVICE` (bind to ports <1024) without needing full root. Best practice: `drop: ["ALL"]` then add back only the 1-2 capabilities actually required.

## 6. NetworkPolicy — Network-Level Isolation

By default, **all Pods can talk to all other Pods** in a cluster — Kubernetes networking is flat and open unless you explicitly restrict it.

```yaml
# Default deny all ingress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
---
# Then explicitly allow only what's needed
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector: {matchLabels: {app: api}}
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector: {matchLabels: {app: frontend}}
    ports: [{protocol: TCP, port: 8080}]
```

- Requires a CNI plugin that supports NetworkPolicy (Calico, Cilium — **not** basic Flannel).
- This is your **microsegmentation** layer — without it, a compromised Pod can freely scan/reach every other Pod in the cluster.

## 7. Tenant / Application Isolation Concepts

For multi-team or multi-tenant clusters, isolation is layered:

```
┌─ Namespace boundary ────────────────────────────────┐
│  ResourceQuota  → prevents one tenant starving others │
│  RBAC (Role+RoleBinding scoped to namespace) → team    │
│    can only touch their own namespace                 │
│  NetworkPolicy (default-deny + explicit allow) →       │
│    tenant pods can't reach other tenants' pods        │
│  Pod Security Admission (restricted) → prevents         │
│    privilege escalation/host access                    │
│  Dedicated ServiceAccounts per app → no shared identity │
└──────────────────────────────────────────────────────┘
```

- **Soft multi-tenancy** (namespaces + the above) is sufficient for trusted internal teams.
- **Hard multi-tenancy** (separate clusters, or virtual clusters like vCluster, or node-level isolation with dedicated node pools + taints) is needed when tenants are mutually untrusted or compliance requires physical separation.

## 8. Hands-On Lab

```bash
kubectl create namespace dev-team
kubectl create serviceaccount ci-deployer -n dev-team

kubectl create role pod-manager \
  --verb=get,list,watch,create,delete --resource=pods -n dev-team
kubectl create rolebinding ci-deployer-binding \
  --role=pod-manager --serviceaccount=dev-team:ci-deployer -n dev-team

# Test what the SA can/cannot do
kubectl auth can-i create pods -n dev-team --as=system:serviceaccount:dev-team:ci-deployer   # yes
kubectl auth can-i delete deployments -n dev-team --as=system:serviceaccount:dev-team:ci-deployer  # no
kubectl auth can-i create pods -n default --as=system:serviceaccount:dev-team:ci-deployer     # no (wrong namespace)

# SecurityContext demo — this should fail if PSA restricted is enforced
kubectl label namespace dev-team pod-security.kubernetes.io/enforce=restricted
kubectl run root-test -n dev-team --image=nginx --overrides='{"spec":{"containers":[{"name":"root-test","image":"nginx","securityContext":{"privileged":true}}]}}'
# Error: violates PodSecurity "restricted"
```

## 10. Installing Security Tooling via Helm

Most CNCF security tools ship only as Helm charts — this is how they're installed in practice:

```bash
# cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# external-secrets operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace

# Kyverno (policy engine)
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
```

**Staff-level point:** for security-critical charts, always pin the chart version
(`--version x.y.z`) and diff CRDs before upgrading — a Helm upgrade can silently
change CRD schemas (e.g., cert-manager) and break existing custom resources if
you don't review the changelog first. Helm does **not** upgrade or delete CRDs on
`helm upgrade` by default — only on a fresh `install`.

## 11. Interview Points
- "RBAC answers 'who can do what' — Role/RoleBinding for namespace scope, ClusterRole/ClusterRoleBinding for cluster scope; ClusterRole+RoleBinding is a common pattern for reusable permission sets applied per-namespace."
- "ServiceAccounts are Pod identities — I always create a dedicated, minimally-scoped SA per app rather than relying on `default`."
- "Kubernetes networking is open by default — NetworkPolicy is required for real isolation, starting with default-deny and adding explicit allows."
- "Defense in depth for multi-tenancy = namespace + quota + RBAC + NetworkPolicy + Pod Security Admission, layered together, not any single control alone."
- "I pin Helm chart versions for security tooling and always check CRD diffs before upgrading — CRD drift is a common source of silent breakage."
