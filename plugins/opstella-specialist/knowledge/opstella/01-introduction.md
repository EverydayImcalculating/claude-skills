# Introduction — Opstella Knowledge

> Source: https://docs.opstella.com/intro/what-is-opstella.html, /opstella-architecture.html, /reference-architecture.html
> Fetched: 2026-05-16
> Coverage: complete (3/3 pages)

## Summary
Opstella is a comprehensive Single Portal for DevSecOps Platform Engineering. It unifies Kubernetes management, CI/CD, security scanning, and observability into one platform so teams don't need to interact with individual tools directly. Use this file for conceptual questions about what Opstella is and how it is structured.

---

## What Is Opstella

Opstella functions as a single portal that orchestrates DevSecOps tooling across Kubernetes environments. It removes the need for developers to directly use GitLab, ArgoCD, Harbor, Vault, etc. — all interactions go through the Opstella UI and the platform handles the tool integrations.

**Tagline:** "Comprehensive Single Portal for DevSecOps Platform Engineering"

---

## Key Features (6 Pillars)

| Feature | Description |
|---|---|
| Platform Management on Multi-Cloud | Manages Kubernetes clusters across cloud providers from a single interface |
| Application and Microservices Management | Deploy, update, and monitor apps and microservices |
| Single Sign-On DevSecOps Tools | Unified SSO (via Keycloak) across all integrated tools |
| Enterprise-Ready DevSecOps Templates | Pre-built CI/CD templates for common tech stacks |
| Centralized Security Dashboard | Aggregates security scan results (SAST, DAST, container scan) |
| Unified and Automated Observability Dashboard | Integrated logs, metrics, and tracing in one view |

---

## Opstella Core Components

Ten internal components that make up the Opstella platform itself:

| Component | Role |
|---|---|
| **UI** | Frontend portal — the web interface users interact with |
| **Core** | Backend service managing data and inter-component communication |
| **Clear Session** | Clears browser cache on SSO authentication |
| **Workers** | Execute integration automation tasks asynchronously |
| **PostgreSQL (Opstella)** | Database for Core service |
| **Redis** | In-memory cache layer |
| **Dapr** | Distributed application runtime — orchestrates component communication |
| **RabbitMQ** | Message broker for async task queuing |
| **Keycloak** | Identity and access management (SSO) |
| **PostgreSQL (Keycloak)** | Dedicated database for Keycloak |

---

## Supported Integrations

### DevOps Tools
- **GitLab** — source code management, CI pipelines
- **ArgoCD** — GitOps-based continuous delivery
- **Harbor** — container image registry
- **Headlamp** — Kubernetes UI

### DevSecOps Tools
- **HashiCorp Vault** — secrets management
- **SonarQube** — static code analysis (SAST)
- **Trivy** — container image vulnerability scanning
- **ZAP (Zed Attack Proxy)** — dynamic application security testing (DAST)
- **DefectDojo** — security findings aggregation and tracking

### Observability Tools
- **Grafana Dashboard** — visualization
- **Grafana Mimir** — long-term metrics storage
- **Grafana Loki** — log aggregation
- **Grafana Tempo** — distributed tracing
- **Grafana Alloy** — telemetry collector (replaces Prometheus Agent/Promtail)

---

## Application Hierarchy (4 Layers)

```
Organization
  └── Platform          (maps to a team / project area; has resource quotas)
        └── Service     (subdivides platform resources; maps to a repo/service boundary)
              └── Component  (a single deployable unit; has its own pipeline + ArgoCD app)
```

| Layer | Managed by | Key concern |
|---|---|---|
| **Organization** | Company admin | Global settings, tool connections |
| **Platform** | Platform admin | Resource quotas, tool selection for the team |
| **Service** | Service owner | Resource subdivision from platform quota |
| **Component** | Developer | Code template, image, deployment config (via OneChart) |

**Rule:** Sum of all Service quotas in a Platform cannot exceed the Platform quota. Sum of Component resources cannot exceed their Service.

---

## Deployment Environment Tiers

Within an Opstella instance, applications are promoted through these tiers:

| Tier | Purpose |
|---|---|
| `dev` | Development — auto-triggered on push to develop branch |
| `sit` | System integration testing |
| `uat` | User acceptance testing |
| `pre` | Pre-production staging |
| `prd` | Production — manual trigger only |

Pipeline promotion: `develop branch` → `MR to main` → `tag v.x.x.x + manual trigger` for prod.

---

## Reference Architecture Variants

### Standalone (PoC / Test environments)

| Resource | Count/Size |
|---|---|
| Nodes | 10 total |
| CPU | 27 cores |
| RAM | 64 GB |
| Disk | 460 GB |

Components: 1 Bastion, 1 HAProxy, 1 NFS, 1 GitLab, 1 Master node, 5 Worker nodes.
Subnet: `192.168.72.0/24`

### Multi-Cluster (Production)

| Resource | Count/Size |
|---|---|
| Nodes | 36 total |
| CPU | 103 cores |
| RAM | 186 GB |
| Disk | 2,140 GB |

Four clusters: DevSecOps (3M+3W), Observability (3M+3W), Non-prod (3M+3W), Prod (3M+5W).
Four distinct subnets: `192.168.72-75.0/24`

**Our setup** matches the multi-cluster variant but uses single-master clusters (demo/dev infra). See `00-local-architecture.md` for exact specs.

---

## Gotchas and Warnings
- Opstella itself is a set of Kubernetes workloads running on the DevSecOps cluster — when the DevSecOps cluster has issues, the entire portal is affected
- Keycloak is the SSO backbone; a Keycloak outage blocks all tool logins through Opstella SSO
- Dapr and RabbitMQ are internal — users never interact with them directly, but their health affects async operations (pipeline triggers, sync jobs)

## Cross-References
- `00-local-architecture.md` — how the multi-cluster reference architecture maps to the actual Deltron host
- `02-roles-and-permissions.md` — the Organization/Platform/Service/Component hierarchy maps directly to the role model
- `03-deploy-applications.md` — how to create Platform/Service/Component and deploy through the hierarchy
