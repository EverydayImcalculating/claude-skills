# Roles and Permissions — Opstella Knowledge

> Source: https://docs.opstella.com/role-and-permissions/ and sub-pages
> Fetched: 2026-05-16
> Coverage: complete (9 pages fetched; role-commendations page content was image-only — partial)

## Summary
Opstella uses a hierarchical RBAC model where roles are assigned at Organization, Platform, Service, or Component layers and automatically inherited downward. Use this file for questions about user access, permission setup, which role to assign, and user creation.

---

## Permission Inheritance Model

Roles cascade **downward** through the hierarchy:

```
Organization (Admin Company)
  └── Platform (Admin)
        └── Service (Full Control / Production / Non-Production / CI-CD Dev / CI-CD Dev Infra)
              └── Component (inherits from Service role)
```

When a user is assigned a role at a higher layer, they receive the same role in all lower layers automatically. Higher-layer roles can do everything lower roles can, plus more.

---

## Role Reference Table

| Role | Layer | Scope | Key Differentiator |
|---|---|---|---|
| **Admin (Opstella)** | Organization | All clusters, all tools | Only role that can delete platforms; full system access |
| **Admin** | Platform | Platform + below | Near-full access; cannot delete platform |
| **Full Control** | Service | Service + below | Full tool access; cannot create/delete platforms |
| **Production** | Service | Service + below | Access to both prod and non-prod Kubernetes; OpenSearch both tenants |
| **Non-Production** | Service | Service + below | Non-prod Kubernetes only; OpenSearch non-prod tenant only |
| **CI/CD Dev** | Component | GitLab only | Pipeline/repo management in GitLab; no Kubernetes access |
| **CI/CD Dev Infra** | Component | GitLab + Non-prod K8s | Like CI/CD Dev but adds non-prod Kubernetes access |

---

## Role Detail: Admin (Opstella) — Organization Layer

The highest role. Assigned only to company account users.

| Tool | Permissions |
|---|---|
| GitLab | Full: repos, branches, tags, webhooks, MRs, CI/CD, members, protected branches, wiki |
| SonarQube | Full: view, manage issues, security hotspots, project settings, user permissions, run analysis |
| Harbor | Full: repos, artifacts, members, permissions, vulnerability scan, replication rules |
| Grafana | View all org charts, create personal dashboards, configure alerts |
| Vault | Full CRUD + list |
| Headlamp | Read-only: pods, services, secrets, deployments, jobs |
| ArgoCD | Full: create/edit/delete apps, sync, actions, metrics |
| Kubernetes | `kube-non-production-admin-role` + `kube-production-admin-role` — full cluster management |

---

## Role Detail: Admin — Platform Layer

Second-tier. Can do almost everything except platform deletion.

Same tool permissions as Admin (Opstella) except: **cannot delete platforms**.

---

## Role Detail: Full Control — Service Layer

Comprehensive service-level access across all tools.

| Tool | Permissions |
|---|---|
| GitLab | Full repo/branch/MR/issue/pipeline/member management |
| SonarQube | Analysis, issues, hotspots, quality profiles, rule adjustment |
| Harbor | Repos, artifacts, members, permissions, scanning, replication |
| Grafana | View, personal dashboards, alerts |
| Vault | Full CRUD + list |
| Headlamp | Read-only (view only, no create/modify) |
| ArgoCD | Full app management, sync, metrics |
| Kubernetes | Both non-prod and prod admin roles |
| OpenSearch | Both non-prod and prod tenants |

---

## Role Detail: Production — Service Layer

| Tool | Key Permissions |
|---|---|
| GitLab | Full repo/branch/MR/issue/pipeline/member management |
| SonarQube | Full analysis + issue/hotspot management + quality profiles |
| Harbor | Full artifact management + scanning + replication |
| Grafana | View + personal dashboards + alerts |
| Kubernetes | `kube-non-production-admin-role` + `kube-production-admin-role` |
| OpenSearch | Both non-prod and prod tenants |

---

## Role Detail: Non-Production — Service Layer

| Tool | Key Permissions |
|---|---|
| GitLab | Full repo/branch/MR/issue/pipeline/member management |
| Kubernetes | `kube-non-production-admin-role` only (no prod access) |
| OpenSearch | Non-prod tenant only |

---

## Role Detail: CI/CD Dev — Component Layer

GitLab-only role. No Kubernetes access.

| Tool | Key Permissions |
|---|---|
| GitLab | Repos, branches, tags, MRs, issues, CI/CD pipelines, member management, protected branches, wiki |
| Kubernetes | **None** |

Best for: developers who write code and manage pipelines but should not touch infrastructure.

---

## Role Detail: CI/CD Dev Infra — Component Layer

CI/CD Dev capabilities plus non-production Kubernetes access.

| Tool | Key Permissions |
|---|---|
| GitLab | Same as CI/CD Dev |
| Kubernetes | `kube-non-production-admin-role` — pods, services, deployments, configmaps, secrets, ingresses, PVCs |

Best for: developers who need to debug pods or inspect cluster state in non-prod.

---

## User Creation

1. Navigate to the **User menu** (admin credentials required)
2. Click **Create User**
3. Fill in:
   - **Username** — cannot be changed after creation (tied to auth system)
   - **Email** — cannot be changed after creation
4. Click **Create User**
5. User appears in User List (may show "Processing" briefly)
6. Assign permissions via **+Permission** button on Platform, Service, or Component layer

**Important:** Username and Email are immutable — they bind to the OIDC authentication system. Get them right at creation.

---

## Permission Assignment Flow

```
1. Create User (org admin)
2. Go to Platform / Service / Component
3. Click +Permission
4. Select User + Role
5. Role inherits downward automatically
```

Cannot modify Opstella Admin user permissions.

---

## Role Recommendations (Image-only page — partial)

> Coverage: partial — the recommendations page content was an image, not extractable text. Fill in manually from the Opstella docs UI.

General guidance derived from role definitions:

| Persona | Recommended Role | Layer |
|---|---|---|
| Platform owner / team lead | Admin | Platform |
| Senior full-stack developer | Full Control | Service |
| Developer with prod deploy rights | Production | Service |
| Developer — non-prod only | Non-Production | Service |
| Developer — code + pipeline only | CI/CD Dev | Component |
| Developer — code + debug non-prod pods | CI/CD Dev Infra | Component |
| Opstella platform admin | Admin (Opstella) | Organization |

---

## Gotchas and Warnings

- Username and Email **cannot be changed** after user creation — they bind to OIDC. Double-check before creating.
- Admin (Opstella) is the **only role that can delete a Platform** — use carefully.
- Headlamp access is **read-only for all roles** — Headlamp is a view-only tool in Opstella context.
- Non-Production role has **no access to prod Kubernetes or prod OpenSearch** — verify before assigning to someone who needs prod debug access.
- Roles cascade **downward only** — assigning a role at Service layer does not give access above the Service level.

## Cross-References
- `01-introduction.md` — the Organization/Platform/Service/Component hierarchy maps to these roles
- `03-deploy-applications.md` — user roles are assigned during Component creation
