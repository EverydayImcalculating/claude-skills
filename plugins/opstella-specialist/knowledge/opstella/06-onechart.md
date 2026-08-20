# OneChart — Opstella Knowledge

> Source: https://docs.opstella.com/onechart/onechart.html
> Fetched: 2026-05-16
> Coverage: complete

## Summary
OneChart is Opstella's universal Helm chart for application deployments. It abstracts raw Kubernetes YAML so developers don't need to know Kubernetes syntax. Every application deployed via Opstella uses OneChart as the Helm chart. Configuration is done through `values.yaml` files.

## Key Concepts
- **OneChart** — a generic Helm chart (`onechart/onechart`) that covers all common deployment patterns
- **values.yaml** — the only file developers need to configure; OneChart renders Kubernetes manifests from it
- **onechart/cron-job** — variant chart for scheduled jobs
- **Sealed Secrets** — encrypted secret values using Bitnami Sealed Secrets operator

---

## Getting Started

```bash
helm repo add onechart https://chart.onechart.dev
helm template my-release onechart/onechart -f values.yaml
```

Minimum required fields: `image.repository` and `image.tag`.

---

## Image Configuration

**Public image:**
```yaml
image:
  repository: nginx
  tag: 1.19.3
```

**Private image (e.g., Harbor / ECR):**
```yaml
image:
  repository: harbor.deltron.local/myproject/myapp
  tag: x.y.z
imagePullSecrets:
  - regcred
```
Note: The `regcred` secret must be pre-created in the cluster namespace.

---

## Environment Variables

```yaml
vars:
  VAR_1: "value 1"
  VAR_2: "value 2"
  DATABASE_URL: "postgres://user:pass@host:5432/db"
```

---

## Secrets Management

### By convention (uses release name as secret name)
```yaml
secretEnabled: true
```
Create the secret beforehand:
```bash
kubectl create secret generic my-release --from-literal=KEY="value"
```

### By custom name
```yaml
secretName: my-custom-secret
```

### File secrets (mounted as files)
```yaml
fileSecrets:
  - name: google-account-key
    path: /google-account-key
    secrets:
      key.json: supersecret
```

### Sealed secrets (encrypted at rest)
```yaml
sealedSecrets:
  secret1: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
```

```yaml
sealedFileSecrets:
  - name: google-account-key
    path: /google-account-key
    filesToMount:
      - name: key.json
        source: AgA/7BnNhSkZAzbMqxMDidxK[...]
```

Seal a value with:
```bash
kubeseal --raw --scope cluster-wide
```
Requires Bitnami Sealed Secrets operator installed in the cluster.

---

## Ingress / Domain Names

**Basic HTTP:**
```yaml
ingress:
  annotations:
    kubernetes.io/ingress.class: nginx
  host: my-app.mycompany.com
```

**HTTPS (manual TLS — references secret `tls-{release-name}`):**
```yaml
ingress:
  host: my-app.mycompany.com
  tlsEnabled: true
```

**HTTPS with Let's Encrypt (cert-manager):**
```yaml
ingress:
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt
  host: my-app.mycompany.com
  tlsEnabled: true
```

**Multiple domains:**
```yaml
ingresses:
  - host: one.mycompany.com
    annotations:
      kubernetes.io/ingress.class: nginx
    tlsEnabled: true
  - host: two.mycompany.com
    tlsEnabled: true
```

---

## Volumes

**Create a new PVC:**
```yaml
volumes:
  - name: data
    path: /data
    size: 10Gi
    storageClass: default
```

**Use an existing PVC:**
```yaml
volumes:
  - name: data
    path: /data
    existingClaim: my-static-claim
```

Common storage classes: `standard` (GCP), `default` (Azure), `do-block-storage` (DigitalOcean), `local-path` (local/RKE2).

---

## Health Checks (Probes)

```yaml
probe:
  enabled: true
  path: "/healthz"
  settings:
    initialDelaySeconds: 0
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 3
    failureThreshold: 3
```

| Field | Description |
|---|---|
| `initialDelaySeconds` | Delay before first probe |
| `periodSeconds` | How often to probe |
| `successThreshold` | Consecutive successes to mark ready |
| `timeoutSeconds` | Probe timeout |
| `failureThreshold` | Failures before marking unready |

---

## High Availability

```yaml
replicas: 2
podDisruptionBudgetEnabled: true
spreadAcrossNodes: true
```

When `replicas > 1`, OneChart automatically adds PodDisruptionBudget and pod anti-affinity rules to spread across nodes.

---

## Resources

```yaml
resources:
  limits:
    cpu: "2000m"
    memory: "2000Mi"
  requests:
    cpu: "200m"
    memory: "500Mi"
```

Always set both `requests` and `limits`. Omitting `requests` causes the pod to use `limits` as requests.

---

## Custom Commands

```yaml
image:
  repository: debian
  tag: stable-slim
command: |
  while true; do date; sleep 2; done
```

Override shell:
```yaml
shell: "/bin/bash"   # default is /bin/sh
# or for Alpine:
shell: "/bin/ash"
```

---

## Security Context

**Container level:**
```yaml
securityContext:
  readOnlyRootFilesystem: true
  runAsNonRoot: true
```

**Init containers:**
```yaml
initContainers:
  - name: init-db
    image: busybox:latest
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
```

---

## Cron Jobs

Use the `onechart/cron-job` chart variant:
```yaml
schedule: "*/5 * * * *"
command: |
  echo "hello from cron"
```

```bash
helm template my-release onechart/cron-job -f values.yaml
```

---

## Prometheus Monitoring Rules

```yaml
prometheusRules:
  - name: KubePodCrashLooping
    message: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is restarting"
    runBookURL: myrunbook.com
    expression: "rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0"
    for: 1h
    labels:
      severity: critical
```

Requires Prometheus Operator (kube-stack-prometheus) in the cluster.

---

## Sidecars

```yaml
sidecar:
  repository: debian
  tag: stable-slim
  shell: '/bin/bash'
  command: 'while true; do sleep 30; done;'
```

Sidecars share the pod's volumes — useful for debug containers or log shippers.

---

## Advanced: Custom Kubernetes Fields

**Pod spec level:**
```yaml
podSpec:
  hostNetwork: true
```

**Container level override:**
```yaml
container:
  imagePullPolicy: Always
```

Use these to pass Kubernetes fields that OneChart doesn't have a dedicated key for.

---

## Gotchas and Warnings

- `secretEnabled: true` expects a secret named exactly after the Helm release. If the secret is missing, the pod will fail to start with `secret not found`.
- `tlsEnabled: true` without a cert-manager annotation expects a pre-existing TLS secret named `tls-{release-name}`. If missing, ingress will serve without TLS silently.
- `replicas: 1` with `podDisruptionBudgetEnabled: true` will block node drains — always use `replicas >= 2` when enabling PDB.
- `readOnlyRootFilesystem: true` breaks apps that write temp files to `/tmp`. Add a `volumes` entry with an emptyDir if needed.
- Sealed secrets are cluster-scoped (`--scope cluster-wide`) — the encrypted value is only valid on the cluster it was sealed against.

## Cross-References
- `04-use-cases.md` — practical OneChart usage examples in Opstella UI
- `03-deploy-applications.md` — how OneChart fits into the CD pipeline (ArgoCD + Helm + OneChart triangle)
