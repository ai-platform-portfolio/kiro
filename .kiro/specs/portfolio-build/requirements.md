# Portfolio Build — Requirements

## Goal

Build a production-grade AI-native platform portfolio targeting senior platform/distributed
systems engineering roles at AI-native companies (reference: Model ML — $87M raised, one of
the fastest-growing enterprise AI platforms, under 18 months from stealth to Series A).

The portfolio must demonstrate:
- Designing control planes and reconciliation systems, not just provisioning infrastructure
- GitOps fleet management that scales to N clusters without N copies of config
- Federated, failure-isolated reconciliation architecture
- AI-native platform tooling (LLM observability, AI code quality gates)
- Reusable, opinionated infrastructure modules with clear interfaces
- Operational depth: lifecycle management, failure handling, bootstrap sequencing

## Repos

### 1. `terraform` — Reusable module library
Reusable Terraform modules with no org-specific values. Opinionated defaults, escape hatches
via optional variables. Called by `bootstrap`.

Modules required:
- `network` — VNet, subnets, NSGs, route tables + associations, private endpoints (for_each)
- `aks-cluster` — AKS cluster, node pools, OIDC issuer, workload identity
- `managed-identity` — user-assigned identity, federated credentials (for_each)
- `key-vault` — Key Vault, access policies, optional private endpoint
- `blob-storage` — storage account, containers (for_each), optional private endpoint
- `dns-zone` — private DNS zone, VNet links (for_each), optional use of existing zone
- `argocd-bootstrap` — Helm release for ArgoCD, namespace, outputs cluster connection info

### 2. `bootstrap` — Infrastructure provisioning
Calls `terraform` modules to provision the portfolio infrastructure:
- Hub-and-spoke networking (management + dev + prod VNets, peering)
- Three AKS clusters (management, dev, prod)
- Managed identities with federated credentials
- Key Vault, Blob Storage, DNS zones
- ArgoCD Helm install on management cluster
- Cluster registration secrets applied to hub ArgoCD

### 3. `gitops` — ArgoCD fleet management (the nucleus)
Federated ArgoCD: hub on management cluster manages Argo agents on workload clusters.
All platform packages deployed via ApplicationSets.

Platform packages:
- cert-manager
- external-secrets (+ Azure Key Vault store)
- external-dns
- envoy-gateway
- LGTM (Loki, Grafana, Tempo, Mimir)
- OpenTelemetry Collector
- Langfuse (LLM observability)
- ClickHouse + Keeper
- CloudNativePG + PgBouncer + Barman
- SonarQube Developer (AI code quality gate)
- ArgoCD agents (Argo-on-Argo, deployed to workload clusters by hub)

### 4. `kiro` — AI session context (this repo)
Steering files for Kiro sessions. Architecture decisions, conventions, network topology,
build plan. Also serves as a living ADR for portfolio reviewers.

### 5. `.github` — Org profile
Public org README, repo catalogue, shared GitHub Actions workflows.

### 6. `app` — Minimal application monorepo (later)
Thin FE + BE to demonstrate platform-layered applications. Not a software engineering
showcase — minimal enough to show the platform works, not more.

## Non-requirements

- Multi-cloud (Azure/AKS only for now)
- Deep software engineering in `app`
- Production secrets or live infrastructure costs beyond a dev cluster
