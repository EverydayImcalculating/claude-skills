# TroubleShoot — Opstella Knowledge

> Source: https://docs.opstella.com/troubleshoot/* (16 pages)
> Fetched: 2026-05-16
> Coverage: complete (16/16 pages)

## Summary
Organized by symptom rather than doc page. Covers pipeline failures, pod issues, sync failures, application access issues, and how to check Opstella system health. Use this file first when something is broken — it maps symptoms to causes and resolution steps, plus includes diagnostic commands.

---

## Diagnostic First Steps

### 1. Check Opstella System Status

Access: `https://<backend-domain>/healthcheck/`

Color coding on the Opstella status page:
- **Green** — tool is operational
- **Red** — tool is not functional

**When a tool shows Red:**

*Config error path:*
1. Access backend admin panel → DevOps tools → find the red-flagged tool
2. Correct config values → save
3. Port-forward to the worker pod for that tool
4. Append `/healthcheck` to trigger a status update

*Pod malfunction path:*
1. Delete the worker pod and affected DevOpsTool pod from Kubernetes (they auto-recreate)
2. Port-forward to the recreated worker pod
3. Append `/healthcheck` to update status

### 2. Check Application Job (Pub/Sub)

When Opstella performs an action (create platform, sync, deploy), it sends a Pub/Sub message to the DevOps tool. If the tool is unavailable and the callback fails, the action stays stuck.

**Resolution:**
1. Access the Django backend admin panel
2. Navigate to the **tasks** section
3. Find the stuck task — identify by **REQUEST SLUG** and **METHOD**
4. Select a worker → click **Send To Pubsub Core-Event**
5. Manually resend the Pub/Sub message

---

## Pipeline Failures

### Wrong Pipeline File Name

**Symptom:** Pipeline does not appear or fails to start at all.
**Cause:** The CI config file is misnamed.
**Resolution:** The file must be named exactly **`.gitlab-ci.yml`** (not `.yaml`, not `.gitlab-ci.yaml`).

### Pipeline Not Using Opstella Template

**Symptom:** Pipeline runs but stages don't match expected Opstella structure.
**Cause:** The `.gitlab-ci.yml` is not referencing the Opstella pipeline template.
**Resolution:** Add the `include` directive pointing to `opstella-cicd/pipeline-template` repository, then re-run.

### Build Stage Fails (Kaniko / Docker)

**Symptom:** Pipeline fails at the **Build** stage with file path errors in logs.
**Cause:** Missing source files; Kaniko cannot find files referenced in the Dockerfile.

**Resolution:**
1. Scroll to bottom of build stage logs to find the exact error
2. Test locally first:
   ```bash
   docker build -t component .
   ```
3. If local build **fails** → fix source code
4. If local build **succeeds** but pipeline fails → files are missing from the Git repo; add them and push

### Image Pull via Proxy Fails

**Symptom:** Build fails downloading external libraries (e.g., `dl-cdn.alpinelinux.org`, npm packages).
**Cause:** Network proxy blocks external domain requests during Kaniko build.

**Resolution:**
1. Open Component Detail → GitLab (via SSO)
2. In GitLab: **CI/CD** → **Variables**
3. Add variable: `KANIKO_EXTRA_ARGS` with proxy configuration
4. Add to Dockerfile:
   ```dockerfile
   ARG SSL_CERT_DIR
   ARG NODE_OPTIONS
   ENV SSL_CERT_DIR=$SSL_CERT_DIR
   ENV NODE_OPTIONS=$NODE_OPTIONS
   ```
5. Re-run pipeline

### Secret Not Mounted (Pod Stays Pending)

**Symptom:** Pod remains in `Pending` state; logs show mount failure.
**Cause:** Secret referenced in Helm values does not exist in the namespace.

**Resolution:**
1. Create the missing secret in the **same namespace** as the pod:
   ```bash
   kubectl create secret generic <secret-name> -n <namespace> --from-literal=KEY=value
   ```
2. Verify namespace matches exactly
3. ArgoCD will sync and pod should start

### Deprecated Library (Node.js Module Not Found)

**Symptom:** Build fails with **"Module not found"** error.
**Cause:** A dependency in `package.json` is deprecated or its version is no longer available.

**Resolution:**
1. Update dependency versions in `package.json`
2. Remove obsolete packages
3. Re-run pipeline

### Network Lost During Pipeline

**Symptom:** Pipeline fails downloading `node_modules` or other packages mid-run.
**Cause:** Temporary network disruption during pipeline execution.

**Resolution:** Click **Retry Pipeline** — the failure is transient. If it fails repeatedly, investigate network connectivity from the GitLab runner node.

### SonarQube Scan Reports Issues

**Symptom:** Pipeline completes but SonarQube alerts appear (bugs, code smells, vulnerabilities).
**Cause:** Code quality issues detected — this is expected behavior, not a scan failure.

**Resolution:**
1. Click the alert notification to view the detailed scan report
2. Review each issue with its description and suggested fix
3. Fix issues in source code
4. Re-run pipeline to verify scan passes

**Note:** If the scan **stage itself fails** (not the quality gate), check SonarQube connectivity from the runner and verify the `SONAR_TOKEN` variable is set in GitLab CI variables.

---

## Pod Issues

### Diagnostic Command

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Also check ArgoCD Application Details for resource status.

### Pod Scheduling Failures (Pending)

**Symptoms:** Pod stuck in `Pending`; describe shows `Insufficient cpu` or `Insufficient memory`.
**Causes:**
- Resource requests/limits too high for available node capacity
- Too many replicas configured
- Nodes at capacity

**Resolution:**
1. In Opstella → Component Detail → Helm Value → reduce `resources.requests.cpu/memory` or `replicas`
2. Or: Opstella → Edit Service → increase the Service resource allocation
3. Check node capacity: `kubectl describe nodes | grep -A5 "Allocated resources"`

### Image Pull Failures (ImagePullBackOff / ErrImagePull)

**Symptom:** Pod in `ImagePullBackOff` or `ErrImagePull`.
**Causes:**
- Wrong image name or tag in Helm values
- Wrong registry URL
- `imagePullSecrets` missing or misconfigured

**Resolution:**
1. Verify image name, tag, and repository in Component's Helm values
2. Check that the `imagePullSecrets` references an existing secret in the namespace:
   ```bash
   kubectl get secrets -n <namespace>
   ```
3. Delete the pod to trigger recreation after fix

### Dependency / Mount Failures

**Symptom:** Pod in `Init:Error` or stays `Pending`; describe shows:
`MountVolume.SetUp failed for volume [name]: [resource type] not found`

**Causes:** Missing ConfigMap, Secret, PVC, or another resource the pod depends on.

**Resolution:**
1. Identify the missing resource type from the error message
2. Create it in the **same namespace**:
   ```bash
   kubectl create configmap <name> -n <namespace> --from-literal=key=value
   kubectl create secret generic <name> -n <namespace> --from-literal=key=value
   ```
3. Re-deploy

---

## Sync Failures

### When to Sync vs When to Escalate

Sync operations fix **data discrepancies** between Opstella's database and the DevOps tool state. They are needed when an action was interrupted mid-way (e.g., DevOps tool was down during Platform creation).

Do NOT sync routinely — only when discrepancies are detected.

### Sync Platform

**Symptom:** Platform data in Opstella doesn't match what's in the DevOps tools (GitLab groups, Harbor projects, ArgoCD projects out of sync).

**Resolution:**
1. App Inventory → select Platform → options button → **Sync Platform**
2. Confirm in the popup
3. Monitor sync progress

Note: Syncing a Platform automatically syncs all Services and Components beneath it.

### Sync Service

**Symptom:** Service-level data discrepancy (GitLab subgroup, quota settings mismatch).

**Resolution:**
1. App Inventory → Platform → Service → options button → **Sync Service**
2. Monitor progress

Note: Syncing a Service automatically syncs all Components within it.

### Sync Component

**Symptom:** Component data discrepancy (GitLab repo, ArgoCD app, Harbor repo out of sync with Opstella).

**Resolution:**
1. App Inventory → Platform → Service → Component → options button → **Sync Component**
2. Monitor progress

### Sync User

**Symptom:** User permissions in a DevOps tool don't match what Opstella shows (e.g., user lost GitLab access).

**Resolution:**
1. Users menu → select user → options button → **Sync User**
2. Monitor processing status

---

## Application Access Issues

### Error 404 — Ingress Not Created

**Symptoms:**
- App URL returns 404
- ArgoCD shows ingress resource failed to sync

**Causes:**
1. Wrong domain (typo in Helm values)
2. Ingress name contains non-English characters

**Resolutions:**

*Wrong domain:*
- Review Helm values → correct the `ingress.host` field → save → sync

*Non-English ingress name:*
- Change `ingress.host` or component name to use only English characters
- Non-English characters in hostnames prevent ArgoCD from creating the ingress resource

### Error 502 Bad Gateway

**Symptom:** App URL loads but returns 502.
**Cause:** Port mismatch — the ingress/service port doesn't match the port the container actually listens on.

**Resolution:**
1. Check the actual port the pod is listening on:
   ```bash
   kubectl exec -n <namespace> <pod-name> -- netstat -tupln
   ```
2. Compare with the `containerPort` in Helm values
3. Update `containerPort` in Helm values to match the actual port
4. Save → ArgoCD syncs → verify access

---

## Escalation Path

| Situation | Owner | Action |
|---|---|---|
| DevOps tool (GitLab/Harbor/ArgoCD) pod crashed | Infra / DevSecOps | Restart pod, check logs, check node resources |
| Opstella worker/core pod crashed | Opstella Dev Team | Notify, provide pod logs |
| Keycloak SSO not working | Infra / DevSecOps | Check Keycloak pod, Keycloak DB, certificate |
| User can't login to Opstella | Check SSO → Keycloak → LDAP/OIDC | If Keycloak OK → Opstella Dev Team |
| Platform/Service/Component creation stuck | DevSecOps | Check Django job task, resend Pub/Sub |
| Pipeline template changed and broke all pipelines | Opstella Dev Team | Notify; rollback template version |
| ArgoCD OutOfSync despite correct git state | DevSecOps | Check ArgoCD app config, namespace, CRDs |

---

## Diagnostic Command Reference

```bash
# Pod status and resource usage
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl top pods -n <namespace>

# Pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous   # if pod restarted

# Check what port a pod is actually listening on
kubectl exec -n <namespace> <pod-name> -- netstat -tupln

# Check secrets in a namespace
kubectl get secrets -n <namespace>

# Check node resource capacity
kubectl describe nodes | grep -A5 "Allocated resources"

# ArgoCD app status
argocd app get <app-name>
argocd app sync <app-name>

# Check Helm release values
helm get values <release-name> -n <namespace>
```

---

## Gotchas and Warnings

- Always check the **Opstella status page** (`/healthcheck/`) before diving into pod/pipeline debug — a red tool means nothing will work regardless of your fix
- Sync operations are for **Opstella↔DevOps tool discrepancies**, not for ArgoCD sync — these are different
- Non-English characters in ingress hostnames silently prevent ingress creation — ArgoCD shows OutOfSync without a clear error
- Pod `Pending` due to resource starvation and `Pending` due to missing secret look similar — always `kubectl describe pod` to distinguish
- `502 Bad Gateway` is almost always a port mismatch — check `containerPort` in Helm values vs actual pod port with `netstat -tupln`

## Cross-References
- `04-use-cases.md` — ArgoCD UI navigation to check sync status and pod health
- `03-deploy-applications.md` — correct deployment setup that prevents most of these issues
- `07-runbook.md` — real cases encountered and resolved in this specific environment
