# Use Cases — Opstella Knowledge

> Source: https://docs.opstella.com/usecase/* (14 pages)
> Fetched: 2026-05-16
> Coverage: complete (14/14 pages)

## Summary
Practical how-to workflows for daily Opstella usage: navigating ArgoCD, checking status, cloning apps, managing templates, running pipelines, updating deployments, monitoring, and configuring secrets. Use this file for step-by-step "how do I do X in Opstella" questions.

---

## ArgoCD UI Navigation

### Manual Menu (per Application)

| Button | Action |
|---|---|
| **Detail** | View and configure application settings |
| **Automated** | Enable auto-sync with GitOps (every 3 minutes) |
| **Prune Resource** | Auto-remove resources when GitOps config removes them |
| **Self-Heal** | Revert any manual `kubectl` changes back to GitOps state |
| **Sync** | Immediately sync with GitOps |
| **Sync Status** | View synchronization history |
| **Refresh** | Reload the page |
| **Delete** | Remove the ArgoCD application |
| **Diff** | Not available for multi-resource apps |
| **History and Rollback** | Not available for multi-resource apps |

### Health Status

| Status | Meaning |
|---|---|
| **Healthy** | Deployment completed successfully |
| **Progressing** | Deployment in progress after sync |
| **Degraded** | At least one resource failed to deploy |
| **Suspended** | Synchronization temporarily halted |
| **Unknown** | Unidentifiable issues detected |

### Sync Status

| Status | Meaning |
|---|---|
| **Synced** | Operation successful — Kubernetes matches GitOps |
| **OutOfSync** | Kubernetes state differs from GitOps config |
| **Unknown** | Configuration or unidentifiable problems |

### View Types

| View | Shows |
|---|---|
| **Tree** | Services, ingress, and deployments in a tree |
| **Pods** | Only deployed pods |
| **Network** | Ingress routing → services → deployments |
| **List** | All resources in table format |

### Filter Options
Filter by: name, resource type, sync status, operational status.

### Chart Actions (per resource)
View logs, sync individual resource, delete resource, open terminal.

---

## Check Application Status

**Method 1 — via kubectl:**
1. Get the Component **Slug** from the Component Detail page
2. Use the Slug to find the pod: `kubectl get pods -n <namespace> | grep <slug>`
3. Check pod status and logs

**Method 2 — via ArgoCD:**
1. Open Component Detail → scroll to DevOps tools → click **ArgoCD**
2. Select the environment to check
3. Login via OIDC
4. Click the icon in the upper right → view Pod info and Application health status

---

## Clone Application

1. Go to **App Inventory** menu
2. Select the Platform → Service → expand component list
3. Click the options button on the target component → select **Clone**
4. In the dialog: choose destination Platform and Service, enter new component name
5. Confirm — Opstella creates the new component

**Important after cloning:**
- The cloned component has a new **Slug** (regenerated automatically)
- Helm values initially reflect the **source** component's config — update them before running pipelines
- Source code in GitLab is also copied

---

## Template Management

### Global Template (Organization-wide)
Available to all Platforms in the Organization.

1. Login to GitLab → Groups → "Your groups"
2. Search for group named **"global template"**
3. Create a new project (blank) in that group
4. Create a branch — branch name = template version identifier
5. Push source code to the branch
6. Add `logo.svg` to main branch for template branding in Opstella UI
7. In Opstella → Create Component → click **sync** button
8. The template appears in the component creation list

### Platform Template (Platform-scoped)
Available only within a specific Platform.

1. Login to GitLab → Groups → "Your groups"
2. Search for your company/platform group name
3. Create a new project (blank) in that group
4. Create a branch, push source code, add `logo.svg` to main
5. Sync in Opstella Component creation page

**Key difference:** Global templates live in a system-level GitLab group visible to all Platforms. Platform templates live in the Platform's own GitLab group.

---

## Manage Pipeline

### Pipeline Structure
Opstella pipelines use the `include` directive in `.gitlab-ci.yaml` to reference templates from `opstella-cicd/pipeline-template` repository.

Template file structure:
- `include` statements — references to shared job definitions
- `stages` — execution order
- `process definitions` — conditional job execution (rules)

### Key `.gitlab-ci.yaml` Fields

| Field | Purpose |
|---|---|
| `stage` | Assigns job to an execution phase |
| `extends` | Inherits job definition from a template |
| `rules` | Conditional logic: when/if the job runs |
| `when` | `always`, `never`, `on_success` |
| `allow_failure` | Allow pipeline to continue even if this job fails |

### Trigger Rules
- Branch-based: develop, main, release branches
- Merge request conditions: source/target branch matching
- Tag-based: `v.x.x.x` semantic versioning pattern
- Manual: `when: manual`

### To Modify a Pipeline
1. Find `.gitlab-ci.yaml` in the component's GitLab repo
2. Edit the file or the referenced template job file
3. Push to trigger a re-run
4. Watch the pipeline stages in GitLab

---

## Update Application

### Edit Existing Helm Values
1. Open **Component Detail** in Opstella
2. Navigate to **Helm Value** section
3. Modify values in the editor
4. Save — Opstella auto-commits to the GitLab config repo
5. ArgoCD syncs within 3 minutes (or click **Sync** for immediate)

### Add New Helm Configuration
1. Click **New Helm** button in Component Detail
2. Enter: `name` (identifies the values file) + Helm values YAML
3. Preview the resulting values
4. Press **Save** — auto-commits to GitLab
5. Go to **Service Detail** → click **Sync component**
6. Verify in ArgoCD (use **Refresh** if needed)
7. Confirm deployment in Component Detail

---

## Using OneChart in Opstella

Real-world example values from Opstella-managed component:

```yaml
nameOverride: component-develop
fullnameOverride: consultant-cloud2-service-component-develop
replicas: 1
image:
  repository: registry.demo2.opstella.in.th/consultant-cloud2-service-component/component
  tag: develop-7232e482
  pullPolicy: Always
imagePullSecrets:
  - image-consultant-cloud2-service
containerPort: 3000
resources:
  requests:
    cpu: 10m
    memory: 10Mi
  limits:
    cpu: 500m
    memory: 500Mi
ingress:
  tlsEnabled: true
  secretName: wildcard-cert-opstella-tls
  host: consultant-cloud2-service-component.dev.demo2.opstella.in.th
  ingressClassName: nginx
```

Key notes:
- `fullnameOverride` fully replaces the release name in resource identifiers — use this to keep resource names predictable
- `imagePullSecrets` must reference a secret that already exists in the namespace
- `tag` is auto-updated by the CI pipeline's `update-gitops-image-tag` stage

---

## Monitoring

All monitoring goes through **Grafana**, accessed via SSO from the Component Detail page.

### Logs (Grafana Loki)
1. Login to Grafana via Component page → OIDC
2. Switch organization → choose org matching your service name in Opstella
3. Dashboard menu → select Log dashboard
4. Filter by **service name**

### Metrics (Grafana Mimir)
1. Login to Grafana via Component page → OIDC
2. Switch organization → choose org matching your service name
3. Dashboard menu → select Metrics dashboard
4. Three filters available:
   - **(1) Cluster** — which cluster to view
   - **(2) Namespace** — environment namespace
   - **(3) Time range** — data period

### Tracing (Grafana Tempo)
1. Login to Grafana via Component page → OIDC
2. Switch organization
3. Dashboard menu → select Tracing dashboard
4. Filter by **service name** and **span name**
5. View Duration metrics and trace details

---

## OpenSearch Index

For log search and dashboard creation.

### Create Index Pattern
1. Access OpenSearch via DevOps tools
2. Left sidebar → **Stack Management** → **Index Patterns**
3. Click **Create index pattern**
4. Enter pattern (must match ≥1 existing index)
5. Select primary time field → confirm

### Use Discover
1. Left menu → **Discover**
2. Filter with: field name queries, time range, additional field filters
3. Execute via **DQL** (Dashboards Query Language)

### Create Dashboard
1. Left menu → **Dashboard** → **Create Dashboard**
2. Name it → **Create**
3. **Add Visualization** → select or create new
4. Choose: data source index, conditions, time range
5. Visualize menu: set query, timeframe, graph type
6. **Update chart** → preview → **Save**
7. Return and add to dashboard

---

## Config Application (Vault Secrets)

Vault is the secrets backend. Access it from the SSO menu or Component Detail.

### Access Vault
**Via SSO menu:**
1. SSO menu → select Platform (top right) → click **Vault**
2. Choose secret engine or secret menu

**Via Component Detail:**
1. App Inventory → Platform → Service → Component → Detail
2. Scroll to **SSO of Component** → click **Vault**

### Add File Secret (mounted as a file in pod)
1. Vault → secret menu → **Create new version**
2. Key = filename, Value = file contents → **Save**
3. In Opstella → Component Detail → Helm Value:
   ```yaml
   fileSecrets:
     - name: <component-name>
       path: <pod-mount-path>
       subPath: <filename>
   ```
4. Save — Opstella commits → ArgoCD auto-redeploys

### Add Environment Variable Secret
1. Vault → secret menu → **Create new version**
2. Key = env var name, Value = secret value → **Save**
3. In Opstella → Component Detail → Helm Value:
   ```yaml
   secretName: <component-name>
   ```
4. Save — ArgoCD auto-redeploys

---

## OpenTelemetry (Tracing Setup)

For Python/FastAPI applications:

**Required libraries:**
```
opentelemetry-distro==0.46b0
opentelemetry-exporter-otlp==1.25.0
opentelemetry-instrumentation-fastapi==0.46b0
opentelemetry-instrumentation-logging==0.46b0
```

**Required environment variables:**
```
OTEL_EXPORTER_OTLP_ENDPOINT=<alloy-endpoint>
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_LOGS_EXPORTER=otlp
OTEL_METRICS_EXPORTER=none
OTEL_PYTHON_LOG_CORRELATION=true
OTEL_RESOURCE_ATTRIBUTES=<service-name-for-tracing>
```

**Query traces in Grafana:**
1. Access Grafana via Component page → OIDC
2. **Explore** menu → Search filters
3. View Duration metrics and trace details

---

## Gotchas and Warnings

- ArgoCD **Self-Heal** will revert any manual `kubectl edit/patch/delete` on managed resources — enable only when you want strict GitOps enforcement
- **Prune Resource** auto-deletes Kubernetes resources removed from GitOps — enable carefully in production
- After cloning a component, the Helm values point to the **original** image and config — update before first pipeline run or the wrong image may deploy
- ArgoCD auto-sync fires every **3 minutes** — if you need immediate deployment, click **Sync** manually
- Grafana organization must match your Opstella service name exactly — wrong org selection shows empty dashboards

## Cross-References
- `03-deploy-applications.md` — the deployment process that produces what you monitor here
- `05-troubleshoot.md` — when monitoring shows issues, go here
- `06-onechart.md` — full reference for all OneChart values shown in use cases
