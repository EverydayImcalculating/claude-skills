# Deploy Applications — Opstella Knowledge

> Source: https://docs.opstella.com/deploy-application/* (8 pages)
> Fetched: 2026-05-16
> Coverage: complete (8/8 pages)

## Summary
Covers the full lifecycle of deploying an application through Opstella: creating the Platform/Service/Component hierarchy, connecting to GitLab, running CI pipelines, checking code quality, managing container images, and deploying via ArgoCD + OneChart. Use this file when someone needs to onboard a new application or understand the deployment flow.

---

## Prerequisites

Before deploying, these must exist:
- An Opstella Organization with at least one admin user
- At least one Kubernetes cluster registered in Opstella
- GitLab, Harbor, ArgoCD, Vault, SonarQube connected to Opstella

---

## Step 1: Create Platform

A Platform is the top-level resource allocation unit for a team or project.

**Required fields:**
- Name
- Resource quotas (CPU, Memory, Storage)
- DevOps tool selection (which GitLab group, Harbor project, ArgoCD project to bind)
- Assigned environments (which Kubernetes clusters / tiers: dev, sit, pre, prd)

**Rule:** The sum of all Service quotas inside this Platform cannot exceed the Platform quota.

---

## Step 2: Create Service

A Service subdivides Platform resources for a logical service boundary (e.g., one microservice team).

**Required fields:**
- Name
- Resource quota (must be ≤ remaining Platform quota)
- Assigned users + roles

---

## Step 3: Create Component

A Component is a single deployable unit with its own pipeline and ArgoCD application.

**Required fields:**
- Name
- Code template selection (determines `.gitlab-ci.yaml` and Helm values structure)
- Resource limits (CPU, Memory) — must fit within Service quota
- Container port
- User role assignments

After creation, Opstella automatically:
- Creates a GitLab repository in the bound group
- Creates an ArgoCD application
- Creates a Harbor repository
- Creates a SonarQube project

---

## Step 4: Clone from Opstella (First Code Push)

1. Open Component detail page in Opstella
2. Scroll to DevOps tools section → click **GitLab**
3. Sign in via "Sign in with Opstella" (SSO)
4. Go to Merge Requests → **New merge request**
5. Select `template` branch → merge into `develop`
6. Complete MR details → **Create merge request**
7. Review diff → Approve if required → **Merge**
8. This triggers the first pipeline automatically
9. Clone the repo locally:
   ```bash
   git clone <SSH URL from GitLab>
   docker build -t api .
   docker run -p 8000:3000 api
   ```

---

## Step 5: Push Existing Code

If you have existing code (not using the template):

1. Select **blank template** when creating the Component
2. In GitLab, only the `develop` branch exists initially
3. Push your code:
   ```bash
   git init
   git remote add origin <your repository url>
   git branch -M main
   git push -uf origin main
   ```

**Required project structure for Opstella CI to work:**

| Item | Purpose |
|---|---|
| `helm/` folder | Helm values files (secrets, env vars) — read by OneChart via ArgoCD |
| `source/` folder | Application source code for pipeline builds |
| `Dockerfile` | Used by Kaniko to build the container image |
| `.gitlab-ci.yaml` | CI/CD pipeline configuration (generated from template or custom) |

---

## Pipeline Workflow (3 Environments)

```
develop branch → Develop pipeline (auto)
      ↓
main branch (MR from develop) → Pre-Production pipeline (auto)
      ↓
Tag v.x.x.x → Production pipeline (manual trigger)
```

| Environment | Trigger | Execution |
|---|---|---|
| **Develop** | Merge to `develop` branch | Automatic immediately |
| **Pre-Production** | Merge `develop` → `main` | Automatic after merge |
| **Production** | Create tag `v.x.x.x` | Manual pipeline trigger required |

---

## CI Pipeline Stages (4 Stages)

| Stage | What It Does |
|---|---|
| **Init** | Prepares variables for subsequent stages |
| **Security** | Source code scanning (SonarQube SAST) |
| **Build** | Builds container image via Kaniko (no Docker daemon); pushes to Harbor |
| **Update-gitops-image-tag** | Updates the image tag in the GitOps config repo; triggers ArgoCD sync |

Tools involved:
- **Kaniko** — builds Docker images inside the pipeline (no Docker daemon required)
- **SonarQube** — SAST scanning during Security stage
- **Trivy** — container image vulnerability scan
- **Harbor** — stores the built image

---

## CD Pipeline (ArgoCD + Helm + OneChart)

The GitOps model:

```
Pipeline updates image tag in config repo (GitLab)
        ↓
ArgoCD detects the change
        ↓
ArgoCD pulls OneChart + reads Helm values from GitLab
        ↓
ArgoCD syncs Kubernetes manifests
        ↓
Kubernetes applies the new deployment
```

Key behaviors:
- ArgoCD continuously monitors for **drift** — any manual `kubectl` change to a managed resource will be reverted to match the Git state
- Helm values live in the `helm/` folder of the Component's repository
- OneChart is the universal Helm chart — no per-app chart needed

---

## Check Code Quality (SonarQube)

1. Open Component detail page → click **SonarQube** link
2. Login via OIDC (SSO)
3. View results after a pipeline has run:

| Category | Meaning |
|---|---|
| **Bugs** | Defects found in code |
| **Vulnerabilities** | Security weaknesses detected |
| **Security Hotspots** | Potential vulnerabilities requiring manual review |
| **Code Smells** | Poor coding practices (technical debt) |
| **Coverage** | Unit test coverage by function execution tracking |

Two views: **New Code** (recent changes) and **Overall Code** (cumulative history).

---

## Manage Registry (Harbor)

1. Open Component detail page → click **Harbor** link
2. Login via OIDC (`LOGIN VIA OIDC PROVIDER`)
3. Browse component's image repository

Each image entry shows:
| Field | Description |
|---|---|
| **Tag** | GitLab commit ID attached to the image |
| **Vulnerability** | Trivy scan result |
| **Push Time** | When the image was uploaded |
| **Pull Time** | When the image was last deployed |

Note: Harbor retention policies are not configured through the Opstella UI — set them directly in Harbor admin.

---

## Resource Quota Rules (Critical)

- Platform quota = ceiling for all services combined
- Service quota = ceiling for all components in that service combined
- Component resource limits must fit within the Service quota
- Opstella enforces this at creation time — you cannot set a Service quota larger than remaining Platform capacity

**Example:** Platform has 10 CPU / 20 GB RAM. Two Services at 5 CPU / 10 GB RAM each fills the Platform. A third Service cannot be created until one is reduced.

---

## Gotchas and Warnings

- ArgoCD is the source of truth for deployment state — **never manually edit Kubernetes resources** in a namespace managed by ArgoCD; changes will be reverted on next sync
- Kaniko builds inside the pipeline pod — if the pipeline pod has resource limits, large Docker builds may OOM and fail
- The `helm/` folder structure must match what OneChart expects — missing required values cause ArgoCD sync to fail with `template error`
- Production pipeline requires **both** a `v.x.x.x` tag AND a manual trigger — tagging alone does not deploy
- If a user's quota at any layer is misconfigured, the pipeline may succeed but the ArgoCD sync will fail due to `LimitRange` or `ResourceQuota` violations in the namespace

## Cross-References
- `01-introduction.md` — Platform/Service/Component hierarchy explained
- `02-roles-and-permissions.md` — user role assignment during Component creation
- `04-use-cases.md` — practical ArgoCD UI navigation, monitoring, and update workflows
- `06-onechart.md` — all OneChart Helm values reference
