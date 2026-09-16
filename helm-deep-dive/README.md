# Helm Deep Dive — Staff-Level Chart Authoring

> Companion track to Days 9, 12, and 14. Those days cover *consuming* charts;
> this covers *authoring and operating* them at the level expected of a 10+ YOE staff engineer.

## Checklist
- [ ] Chart anatomy: Chart.yaml, values.yaml, templates/, charts/, _helpers.tpl
- [ ] Templating: values injection, conditionals, loops, named templates
- [ ] Chart dependencies (subcharts) and conditional dependencies
- [ ] Helm hooks (pre-install, post-upgrade, etc.) and hook weights/deletion policies
- [ ] Library charts vs application charts
- [ ] Values schema validation (values.schema.json)
- [ ] Testing charts (helm lint, helm template, helm unittest, ct/chart-testing)
- [ ] Packaging & publishing (OCI registries vs classic chart repos)
- [ ] Multi-environment values strategy (values-dev/staging/prod.yaml layering)
- [ ] Helm vs Kustomize — when to use which, and why many shops use both together
- [ ] Security: avoiding secrets in values.yaml, chart provenance/signing (helm sign)

---

## 1. Chart Anatomy

```
mychart/
├── Chart.yaml              # metadata: name, version, appVersion, dependencies
├── values.yaml              # default values, the "API" of your chart
├── values.schema.json        # JSON schema to validate values at install time
├── charts/                   # vendored subchart dependencies (.tgz)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── _helpers.tpl          # named templates / helper functions
│   ├── NOTES.txt              # printed after install/upgrade
│   └── tests/
│       └── test-connection.yaml   # helm test
└── .helmignore
```

## 2. Templating Essentials

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{- if .Values.serviceAccount.create }}
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        {{- range .Values.env }}
        env:
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
```

```yaml
# templates/_helpers.tpl
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

**Staff-level habit:** always run `helm template . -f values-prod.yaml` before
`upgrade` in CI, and diff it against the previous rendered output (e.g., via the
`helm-diff` plugin) — this catches unintended changes (image tag drift, accidental
replica count reset) before they hit a cluster.

## 3. Chart Dependencies (Subcharts)

```yaml
# Chart.yaml
dependencies:
- name: postgresql
  version: "13.2.0"
  repository: "https://charts.bitnami.com/bitnami"
  condition: postgresql.enabled     # toggle subchart via values.yaml
- name: redis
  version: "18.0.0"
  repository: "https://charts.bitnami.com/bitnami"
  condition: redis.enabled
```

```bash
helm dependency update      # pulls subcharts into charts/
helm dependency build        # rebuilds from Chart.lock
```

- Subchart values are namespaced under the subchart's name in your `values.yaml`
  (e.g., `postgresql.auth.password`).
- **Tradeoff to know:** bundling infra (DB, cache) as subcharts is convenient for
  demos but usually wrong for production — most staff engineers keep stateful
  infra (databases) managed separately (Terraform/managed service) and use Helm
  only for the stateless application layer.

## 4. Helm Hooks

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["./migrate.sh"]
      restartPolicy: Never
```

- Hooks run **outside** the normal release object tracking — they're not part of
  `helm rollback`. Use them for one-shot tasks (DB migrations, cache warm-up),
  never for anything that needs to be part of the rolled-back state.
- `hook-weight` controls ordering when multiple hooks share a phase.
- `hook-delete-policy` controls cleanup — `before-hook-creation` prevents stale Job
  name collisions on repeated upgrades.

## 5. Library Charts

```yaml
# Chart.yaml for a shared library chart (no templates of its own render directly)
apiVersion: v2
name: common
type: library
version: 1.0.0
```

- A **library chart** provides only named templates (`_helpers.tpl`-style) to be
  imported by other charts via `dependencies` — no standalone resources.
- Staff-level use case: standardize labels, annotations, resource naming, and
  security context defaults across dozens of internal charts/microservices —
  one library chart, many consuming charts, consistent conventions enforced
  centrally instead of copy-pasted.

## 6. Values Schema Validation

```json
// values.schema.json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "replicaCount"],
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" }
      }
    }
  }
}
```

- `helm install`/`upgrade` fails fast with a clear error if `values.yaml` violates
  this schema — critical for a chart consumed by many teams, prevents "typo in
  values.yaml deployed to prod" incidents.

## 7. Testing Charts

```bash
helm lint ./mychart                                    # static analysis
helm template ./mychart -f values-prod.yaml             # render without installing
helm install --dry-run --debug my-release ./mychart     # server-side dry-run (validates against API)
helm test my-release -n production                       # runs templates/tests/*.yaml

# chart-testing (ct) — used in CI for chart repos
ct lint --charts ./mychart
ct install --charts ./mychart
```

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

## 8. Packaging & Publishing

```bash
helm package ./mychart                       # produces mychart-1.0.0.tgz
helm push mychart-1.0.0.tgz oci://my-registry.example.com/helm-charts   # OCI (modern, preferred)

# Classic chart repo (index.yaml based) — older pattern, still common
helm repo index . --url https://charts.example.com
```

- **OCI registries** (ECR, ACR, GHCR, Harbor) are the modern standard — charts are
  just another artifact type alongside container images, same auth/versioning/
  scanning pipeline. Classic `index.yaml`-based repos (GitHub Pages, ChartMuseum)
  are legacy but still widely seen.

## 9. Multi-Environment Values Strategy

```
mychart/
├── values.yaml          # sane defaults / base
├── values-dev.yaml       # overrides: low replicas, debug logging
├── values-staging.yaml
└── values-prod.yaml      # overrides: HA replica count, resource limits, PDB enabled
```

```bash
helm upgrade myapp ./mychart -f values.yaml -f values-prod.yaml -n production
```

- Values files layer left-to-right, **later files win** — this is the Helm
  equivalent of Kustomize overlays.
- **Staff-level practice:** keep `values.yaml` as the only place defaults live;
  environment files should contain *only the deltas* (a handful of keys), not a
  full copy — full copies drift and become impossible to diff meaningfully over time.

## 10. Helm vs Kustomize — When to Use Which

| | Helm | Kustomize |
|---|------|-----------|
| Templating | Full templating language (loops, conditionals, functions) | No templating — patch-based (strategic merge / JSON patch) |
| Packaging/versioning | First-class (Chart.yaml version, OCI push/pull) | No native packaging — just directories in git |
| Third-party software | The standard — virtually every CNCF project ships a Helm chart | Rarely how third-party software is distributed |
| Your own app manifests | Can feel like overkill/complexity for simple apps | Simpler, plain YAML + patches, easier to read/review in PRs |
| Learning curve | Higher (Go templates, subcharts, hooks) | Lower (pure YAML, standard kubectl feature) |

**Common staff-level pattern:** use **Helm to install third-party/vendor software**
(ingress-nginx, cert-manager, Prometheus stack, AWS LB Controller) and **Kustomize
(or plain manifests via GitOps) for your own application deployments** — you get
Helm's ecosystem leverage where it matters and avoid its templating complexity
for code you fully own. Some shops use `helm template` to render a chart once,
then check in the rendered output and manage it with Kustomize overlays from there
("de-helmify" pattern) for maximum auditability.

## 11. Security Practices

- **Never put secrets directly in `values.yaml`** committed to git — reference
  External Secrets / Vault-injected Secrets by name, or use the `helm secrets`
  plugin (SOPS-encrypted values files) if secrets must live in values.
- **Pin chart versions explicitly** (`--version x.y.z`) in CI/CD — never `helm
  install` against a floating "latest" in production pipelines.
- **Verify chart provenance** where available: `helm install --verify` against
  a `.prov` file (chart signed with the maintainer's PGP key).
- **Review CRD changes** before upgrading charts that own CRDs (cert-manager,
  Prometheus Operator) — Helm by design **does not upgrade or delete CRDs**
  automatically on `helm upgrade` (only on fresh `install`), a common source of
  "why isn't my CRD change taking effect" incidents.

## 12. Hands-On Lab — Write and Publish a Chart

```bash
helm create mychart
cd mychart

# Trim the scaffold down, edit values.yaml + templates/deployment.yaml to match
# the ASP.NET Core app from Day 2

helm lint .
helm template . | less
helm install demo . --dry-run --debug
helm install demo .
helm upgrade demo . --set replicaCount=3
helm history demo
helm rollback demo 1
helm uninstall demo

# Package and push to a local OCI registry (if available) or just package locally
helm package .
```

## 13. Interview Points (Staff-Level)
- "Helm's release model (versioned, rollback-able) is what differentiates it from
  Kustomize — it treats a deployed set of resources as a single manageable unit
  with history, not just a YAML rendering step."
- "I use library charts to centralize labeling/security-context conventions across
  dozens of internal charts, so a policy change (e.g., adding a required
  annotation) is a one-line change in one place, not a PR across 40 repos."
- "Helm hooks run outside normal release tracking — I use them for migrations and
  one-shot jobs, never for anything that needs to roll back with the release."
- "A recurring gotcha I watch for: Helm doesn't manage CRD lifecycle on upgrade by
  default — I always check CRD diffs manually for charts like cert-manager or
  kube-prometheus-stack before upgrading."
- "For third-party software I default to Helm; for our own services I lean
  Kustomize or rendered-and-committed manifests — it keeps the diff in a PR
  human-readable instead of hidden behind template logic."
