---
inclusion: always
---

# Conventions

## Terraform modules

- All modules live in `terraform/modules/<name>/`
- Each module has: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `README.md`
- Provider versions pinned to `azurerm ~> 4.0`, `helm ~> 2.0`, `kubernetes ~> 2.0`
- Required variables = things that genuinely differ per deployment (names, CIDRs, IDs)
- Optional variables with defaults = platform team opinions (SKUs, replica counts, flags)
- Private endpoints use `for_each` over a map input — caller passes
  `{ name, resource_id, subresource, subnet_id }` and the module stamps them all out
- Private DNS zones: created by default, accept existing zone IDs via optional variable
  (escape hatch for hub DNS scenarios)
- Resource naming: `var.name` is the base, module appends resource type suffix where needed
  e.g. `"${var.name}-kv"` for Key Vault

## ArgoCD / GitOps

- One ApplicationSet per platform package in `platform/applicationsets/`
- Cluster-specific overrides in `platform/packages/<name>/values/{{cluster}}.yaml`
- Shared defaults in `platform/packages/<name>/values.yaml`
- Cluster identity defined in `clusters/<name>/cluster-values.yaml`
- Sync waves used for dependency ordering (cert-manager before everything that needs certs)
- Health checks on CRDs before dependent resources (prevents dry-run failures on empty cluster)

## Naming conventions

| Resource | Pattern | Example |
|----------|---------|---------|
| Resource groups | `rg-<purpose>-<env>` | `rg-platform-prod` |
| AKS clusters | `aks-<purpose>-<env>` | `aks-management`, `aks-dev`, `aks-prod` |
| VNets | `vnet-<purpose>-<env>` | `vnet-management`, `vnet-dev` |
| Subnets | `snet-<purpose>-<env>` | `snet-aks-system`, `snet-pe` |
| NSGs | `nsg-<subnet>-<env>` | `nsg-aks-system-prod` |
| Route tables | `rt-<vnet>-<env>` | `rt-management` |
| Key Vaults | `kv-<purpose>-<env>` | `kv-platform-prod` |
| Managed identities | `id-<purpose>-<env>` | `id-platform-prod`, `id-postgres-prod` |
| Storage accounts | `st<purpose><env>` (no hyphens, max 24 chars) | `stplatformprod` |
| ArgoCD namespaces | `argocd` on all clusters | |
| Platform namespaces | match package name | `cert-manager`, `postgres`, `lgtm` |

## Authentication

Prefer OAuth over long-standing credentials everywhere — across the portfolio, local tooling,
and MCP integrations:

- **OAuth is the default** for any service that supports it (Linear, GitHub, Azure, etc.)
- **Long-standing API keys or tokens are only acceptable where OAuth is not supported**
- Keys that must exist (e.g. service-to-service where OAuth is unsupported) go in Azure Key
  Vault or equivalent secrets manager — never in config files, environment variables baked
  into images, or Git
- Local developer tooling (MCP servers, CLI auth) follows the same rule — use OAuth device
  flow or browser flow rather than personal API keys where the option exists
- Workload identity / federated credentials (OIDC) are preferred over static keys for
  any Azure service-to-service auth (this is already the pattern in the platform via ESO
  + managed identity)

## Git

- Branch naming: `feat/<scope>`, `chore/<scope>`, `fix/<scope>`
- Commits reference Linear issue IDs where applicable
- `main` is the production branch — Argo syncs from `main`
- Feature branches used for all changes, merged via PR
- No secrets in Git, ever

## Documentation standard

Every module and package has a README that covers:
1. What it does (one paragraph)
2. Usage example (minimal working call)
3. Inputs table (name, type, default, description)
4. Outputs table
5. Any production notes or known simplifications (e.g. DNS hub simplification)
