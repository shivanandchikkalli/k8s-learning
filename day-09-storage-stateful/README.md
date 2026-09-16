# Day 9 — Storage & Stateful Workloads

## Checklist
- [ ] Volumes
- [ ] PersistentVolume (PV)
- [ ] PersistentVolumeClaim (PVC)
- [ ] StorageClass
- [ ] Dynamic provisioning
- [ ] StatefulSet
- [ ] Deployment vs StatefulSet
- [ ] Stable identity and persistent storage
- [ ] Deploy a simple stateful workload locally

---

## 1. Volumes — Pod-Scoped Storage

A `volume` (e.g., `emptyDir`) lives and dies with the **Pod**, not the container — it survives a container restart within the same Pod, but is deleted when the Pod is deleted.

```yaml
volumes:
- name: scratch
  emptyDir: {}          # ephemeral, gone when Pod is deleted
```

## 2. PersistentVolume / PersistentVolumeClaim — Decoupling Storage from Pods

```
PersistentVolume (PV)     = actual piece of storage (cluster-scoped resource,
                             provisioned by admin or dynamically by a StorageClass)
PersistentVolumeClaim(PVC)= a Pod's REQUEST for storage (namespaced) — 
                             "I need 10Gi, ReadWriteOnce, fast-ssd"
Pod                        = mounts a PVC by name, doesn't know/care where it lives
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo hello > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

- The **PV outlives the Pod** — if the Pod is deleted and recreated (or rescheduled to another node), it can reattach to the same PV/data via the same PVC, as long as the access mode allows it.

## 3. StorageClass & Dynamic Provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com     # CSI driver (cloud-specific)
parameters:
  type: gp3
reclaimPolicy: Retain            # keep data even if PVC is deleted
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

- Without dynamic provisioning, an admin would have to manually pre-create PVs to match every PVC request — impractical at scale.
- With a **StorageClass**, when a PVC references it, the cloud provider's CSI driver **automatically creates a matching PV** (e.g., a real EBS volume) on demand.
- `WaitForFirstConsumer` delays provisioning until a Pod using the PVC is actually scheduled — ensuring the volume is created in the correct zone/node topology.

## 4. StatefulSet vs Deployment

| | Deployment | StatefulSet |
|---|-----------|-------------|
| Pod identity | Random suffix, interchangeable (`web-7f9c8d-abc12`) | Stable, ordered (`web-0`, `web-1`, `web-2`) |
| Pod DNS | Not individually addressable | Stable per-pod DNS via headless Service: `web-0.web-svc.ns.svc.cluster.local` |
| Storage | Pods typically share or have no PVC, or all share one | Each Pod gets **its own PVC**, created from a `volumeClaimTemplate`, and reattaches to the SAME PVC if the Pod is rescheduled |
| Scaling order | Parallel, any order | Sequential (0, 1, 2... up; reverse down), by default |
| Use case | Stateless apps (web servers, APIs) | Databases, queues, anything needing stable identity/storage (Postgres, Kafka, ZooKeeper, Elasticsearch) |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless        # required: a headless Service
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:               # one PVC PER POD, auto-created
  - metadata: {name: data}
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources: {requests: {storage: 1Gi}}
---
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None       # headless -> DNS resolves directly to each pod IP
  selector: {app: web}
  ports: [{port: 80}]
```

## 5. Stable Identity and Persistent Storage — Why It Matters

- Each StatefulSet Pod gets a **predictable name** (`web-0`) and **stable DNS** (`web-0.web-headless.default.svc.cluster.local`) that doesn't change even if the Pod is deleted and recreated.
- Its PVC (`data-web-0`) is **not deleted** when the Pod is rescheduled — the replacement `web-0` reattaches to the exact same volume with the exact same data.
- This is essential for anything with **per-replica state** — e.g., a Postgres replica needs to keep its own data directory across restarts; a Kafka broker needs a stable broker ID and its own log segments; losing that mapping would mean data loss or a broken cluster membership.

## 6. Hands-On Lab

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector: {app: web}
  ports: [{port: 80}]
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        volumeMounts:
        - {name: data, mountPath: /usr/share/nginx/html}
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: {requests: {storage: 500Mi}}
EOF

kubectl get pods -l app=web -w &
kubectl get pvc

# Prove identity/data persistence: write a file to web-0, delete the pod, confirm data survives
kubectl exec web-0 -- sh -c "echo persisted-data > /usr/share/nginx/html/index.html"
kubectl delete pod web-0
kubectl wait --for=condition=ready pod/web-0 --timeout=60s
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html   # still "persisted-data"

kubectl get pod web-0 -o jsonpath='{.status.podIP}'
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup web-0.web-headless.default.svc.cluster.local
```

## 7. Helm Basics — Installing & Managing Charts

### Why Helm Exists
Raw YAML doesn't parameterize well across environments (dev/staging/prod) and has no
versioning/rollback story for a *release* as a unit (a Deployment + Service + ConfigMap
+ Secret shipped/rolled-back together). Helm solves this with **charts** (templated
packages) and **releases** (a named, versioned instance of a chart's values).

### Core Workflow

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgresql

helm install my-postgres bitnami/postgresql \
  --namespace database --create-namespace \
  --set auth.postgresPassword=secret \
  --set primary.persistence.size=20Gi

helm list -n database
helm status my-postgres -n database
helm get values my-postgres -n database        # what values are actually in effect
helm get manifest my-postgres -n database       # the rendered YAML that was applied

helm upgrade my-postgres bitnami/postgresql -f values-prod.yaml -n database
helm rollback my-postgres 1 -n database          # roll back to revision 1
helm uninstall my-postgres -n database
```

### Release = State, Not Just Templates
Every `install`/`upgrade` creates a new **revision**, stored as a Secret in the
release's namespace (`sh.helm.release.v1.<name>.v<N>`). This is what makes
`helm rollback` possible — Helm diffs against the last known-good rendered manifest,
not just against your `values.yaml`.

For a full staff-level treatment of chart authoring (templating, hooks, library
charts, testing, OCI publishing), see [helm-deep-dive/README.md](../helm-deep-dive/README.md).

## 8. Interview Points
- "PV/PVC decouples storage lifecycle from Pod lifecycle — the Pod is disposable, the data isn't."
- "StorageClass + dynamic provisioning means developers just request storage by class/size; the actual cloud disk gets created automatically."
- "I use StatefulSet only when I need stable network identity AND stable per-replica storage — for stateless apps, Deployment is simpler and sufficient."
- "Helm's release model gives me versioned, rollback-able deployments of a full set of resources as a unit — that's what plain `kubectl apply` doesn't provide."
