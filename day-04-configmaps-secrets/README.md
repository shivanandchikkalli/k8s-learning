# Day 4 — ConfigMaps & Secrets

## Checklist
- [ ] ConfigMap
- [ ] Secret
- [ ] Environment variables vs mounted configuration
- [ ] External secret-management pattern
- [ ] AWS Secrets Manager → External Secrets → Kubernetes Secret
- [ ] Deploy an application using configuration and secrets
- [ ] Understand encoding vs encryption

---

## 1. ConfigMap — Externalized, Non-Sensitive Config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aspnet-config
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Information"
  appsettings.json: |
    {
      "ConnectionStrings": { "Default": "..." },
      "FeatureFlags": { "NewCheckout": true }
    }
```

## 2. Secret — Sensitive Data

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:            # plain text in, base64 stored in etcd automatically
  username: appuser
  password: S3cur3P@ss
```

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=appuser \
  --from-literal=password='S3cur3P@ss'
```

## 3. Encoding vs Encryption — Critical Distinction

| | Base64 (Secret default) | Encryption |
|---|---|---|
| Reversible without a key? | **Yes** — anyone can `base64 -d` it | No — requires a secret key |
| Purpose | Makes binary-safe data storable as text | Protects confidentiality |
| Kubernetes Secrets by default | Only base64-**encoded**, NOT encrypted | Optional — must enable `EncryptionConfiguration` at the API server so etcd stores ciphertext |

```bash
echo -n "S3cur3P@ss" | base64          # U0AzY3VyM1BAc3M=
echo -n "U0AzY3VyM1BAc3M=" | base64 -d  # S3cur3P@ss  -- trivially reversible!
```

**Takeaway:** A Kubernetes Secret is not secret at rest unless you explicitly enable etcd encryption at rest, AND lock down RBAC access to `secrets`. Anyone who can `kubectl get secret -o yaml` can decode it instantly.

## 4. Environment Variables vs Mounted Configuration

| | Env Vars | Mounted Volume |
|---|---|---|
| Update behavior | Requires Pod restart to pick up changes | File updates automatically (kubelet sync, ~60s) without restart |
| Visibility | Visible in `kubectl describe pod`, process env, crash dumps | Only visible to processes reading the file |
| Best for | Small values, feature flags, simple config | Large configs, files apps expect on disk (appsettings.json, certs), anything needing hot-reload |
| Secrets exposure risk | Higher (env vars leak into logs/child processes easily) | Lower (file permissions control access) |

```yaml
spec:
  containers:
  - name: aspnet-app
    image: mcr.microsoft.com/dotnet/samples:aspnetapp
    envFrom:
    - configMapRef:
        name: aspnet-config
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: aspnet-config
      items:
      - key: appsettings.json
        path: appsettings.json
```

## 5. External Secret-Management Pattern

Native K8s Secrets are weak on their own (no rotation, no audit trail, base64 only). Production systems externalize the source of truth.

```
┌───────────────────┐     ┌────────────────────┐     ┌──────────────────┐
│ AWS Secrets Manager│────▶│ External Secrets    │────▶│ Kubernetes Secret │
│ (source of truth,  │ IAM │ Operator (ESO)      │ CRD │ (auto-created,    │
│  rotation, audit)  │     │ (controller in-clstr)│     │  auto-refreshed)  │
└───────────────────┘     └────────────────────┘     └──────────────────┘
                                                              │
                                                    mounted/env into Pod
```

```yaml
# ExternalSecret CRD — ESO watches this and syncs from AWS
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-credentials      # K8s Secret created/updated by ESO
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: prod/myapp/db
      property: password
```

**Why this pattern:** rotation happens in AWS Secrets Manager (or Vault/Azure Key Vault) without redeploying; access to the real secret store is controlled by IAM, not just K8s RBAC; full audit trail lives outside the cluster.

## 6. Hands-On Lab

```bash
kubectl create configmap aspnet-config \
  --from-literal=ASPNETCORE_ENVIRONMENT=Production

kubectl create secret generic db-credentials \
  --from-literal=username=appuser --from-literal=password='S3cur3P@ss'

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep -E 'ASPNETCORE|DB_' && sleep 3600"]
    envFrom:
    - configMapRef:
        name: aspnet-config
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF

kubectl logs config-demo

# Prove base64 is not encryption
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

## 7. Interview Points
- "Secrets are base64-encoded, not encrypted, by default — encryption at rest requires explicit API server configuration."
- "I mount config as a volume when I need hot-reload without restarting pods; I use env vars for simple, static values."
- "For production, I never store real secrets as native K8s Secrets directly — I sync them from AWS Secrets Manager via External Secrets Operator so rotation and audit live in a proper secret store."
