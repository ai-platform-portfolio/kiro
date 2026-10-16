# Portfolio Build — Design

## Repo build order

```
1. terraform        ← reusable modules, no dependencies
2. bootstrap        ← calls terraform modules
3. gitops           ← ArgoCD fleet config (migration from Flux already in progress)
4. kiro             ← session context (this repo, created first)
5. .github          ← org profile updates as repos are added
6. app              ← last, after platform is proven
```

## terraform — module design

### Module interface pattern

Every module follows the same contract:
- `var.name` — base name, used to derive all resource names
- `var.location` — Azure region
- `var.resource_group_name` — caller manages resource group
- `var.tags` — merged with any module-level tags
- Required variables are truly required (no defaults)
- Optional variables carry opinionated defaults, documented in README

### `network` module

Inputs:
- `name` — base name for VNet and derived resources
- `resource_group_name`, `location`, `tags`
- `address_space` — list of CIDRs for the VNet
- `subnets` — map of `{ cidr, service_endpoints?, delegations? }` keyed by subnet name
- `nsg_rules` — map of subnet name → list of NSG rules (module creates one NSG per subnet)
- `route_table_routes` — list of routes for the single shared route table
- `route_table_subnet_associations` — list of subnet names to associate the route table to
- `private_endpoints` — map of `{ resource_id, subresource, subnet_key, dns_zone_id? }`
- `private_dns_zones` — map of zone names to create (e.g. `"privatelink.vaultcore.azure.net"`)
- `vnet_links` — map of VNet IDs to link to all created DNS zones

Outputs:
- `vnet_id`, `vnet_name`
- `subnet_ids` — map of subnet name → subnet ID
- `nsg_ids` — map of subnet name → NSG ID
- `private_dns_zone_ids` — map of zone name → zone ID
- `private_endpoint_ips` — map of PE name → private IP

DNS note: In production hub-and-spoke, private DNS zones are linked only to the hub VNet
and resolution is forwarded via Azure Firewall DNS proxy. This module links zones directly
to all provided VNets (var.vnet_links) as a simplification for environments without a DNS
server. This is documented in the module README.

### `aks-cluster` module

Inputs:
- `name`, `resource_group_name`, `location`, `tags`
- `kubernetes_version` — pinned, not "latest"
- `network_plugin` — default `"azure"` (Azure CNI)
- `vnet_subnet_id` — system node pool subnet
- `system_node_pool` — `{ vm_size, min_count, max_count, os_disk_size_gb }`
- `user_node_pools` — map of additional node pools (for_each)
- `workload_identity_enabled` — default `true`
- `oidc_issuer_enabled` — default `true`
- `max_pods` — default `110` (explicit, avoids Azure CNI 30-pod default trap)
- `private_cluster_enabled` — default `false` for portfolio, `true` for prod intent
- `authorized_ip_ranges` — for public API server access
- `admin_group_object_ids` — Entra group IDs for cluster admin

Outputs:
- `cluster_id`, `cluster_name`
- `oidc_issuer_url` — used by managed-identity module for federation
- `kubelet_identity` — for role assignments
- `kube_config` — sensitive

### `managed-identity` module

Inputs:
- `name`, `resource_group_name`, `location`, `tags`
- `federated_credentials` — map of `{ name, subject, issuer, audiences? }` (for_each)
  - `subject` is the Kubernetes service account subject:
    `"system:serviceaccount:<namespace>:<sa-name>"`
  - `issuer` is the AKS OIDC issuer URL

Outputs:
- `identity_id`, `client_id`, `principal_id`
- `federated_credential_ids`

### `key-vault` module

Inputs:
- `name`, `resource_group_name`, `location`, `tags`
- `tenant_id`
- `sku_name` — default `"standard"`
- `soft_delete_retention_days` — default `7`
- `purge_protection_enabled` — default `false` (set true for prod)
- `access_policies` — map of `{ object_id, secret_permissions, key_permissions? }`
- `private_endpoint` — optional `{ subnet_id, dns_zone_id }` (null = no PE)
- `network_acls` — optional, default allows all (tighten for prod)

Outputs:
- `vault_id`, `vault_uri`

### `blob-storage` module

Inputs:
- `name`, `resource_group_name`, `location`, `tags`
- `account_tier` — default `"Standard"`
- `account_replication_type` — default `"LRS"`
- `containers` — map of `{ name, access_type? }` (for_each, access_type default "private")
- `private_endpoint` — optional `{ subnet_id, dns_zone_id }`
- `network_rules` — optional, default allows all

Outputs:
- `storage_account_id`, `storage_account_name`
- `primary_blob_endpoint`
- `container_ids` — map of container name → resource ID

### `dns-zone` module

Inputs:
- `name` — the DNS zone name e.g. `"platform.example.com"` or `"privatelink.vaultcore.azure.net"`
- `resource_group_name`, `location`, `tags`
- `zone_type` — `"public"` or `"private"` (default `"private"`)
- `vnet_links` — map of `{ vnet_id, registration_enabled? }` — for_each (private zones only)
- `existing_zone_id` — optional, if provided the module skips creation and uses this zone ID
  (escape hatch for hub DNS scenarios)
- `records` — optional map of DNS records (A, CNAME, TXT)

Outputs:
- `zone_id`, `zone_name`
- `name_servers` — for public zones

### `argocd-bootstrap` module

Inputs:
- `name` — release name, default `"argocd"`
- `namespace` — default `"argocd"`
- `chart_version` — pinned, no "latest"
- `values` — map of Helm values (merged with module defaults)
- `create_namespace` — default `true`
- `cluster_registration_secrets` — map of workload clusters to pre-register
  `{ server, ca_data, bearer_token }` — sensitive

Outputs:
- `release_name`, `namespace`
- `server_url`

## bootstrap — root module design

```
bootstrap/
├── main.tf              # Calls all terraform modules
├── variables.tf         # Env-specific inputs (region, names, sizes)
├── outputs.tf
├── versions.tf          # Provider pins
├── terraform.tfvars.example
└── README.md
```

Single root module (not per-environment workspaces) for portfolio simplicity.
Environment differences handled via tfvars.

## gitops — directory structure

```
gitops/
├── bootstrap/
│   └── management/
│       ├── argocd-values.yaml     # Hub Argo Helm values (referenced by Terraform)
│       └── self-manage.yaml       # ArgoCD Application pointing at its own Helm release
├── platform/
│   ├── applicationsets/           # One ApplicationSet per platform package
│   │   ├── argocd-agents.yaml     # Deploys Argo to workload clusters (Argo-on-Argo)
│   │   ├── cert-manager.yaml
│   │   ├── external-secrets.yaml
│   │   ├── external-dns.yaml
│   │   ├── envoy-gateway.yaml
│   │   ├── lgtm.yaml
│   │   ├── opentelemetry.yaml
│   │   ├── langfuse.yaml
│   │   ├── clickhouse.yaml
│   │   ├── postgres.yaml
│   │   └── sonarqube.yaml
│   └── packages/                  # Helm values per package
│       ├── <package>/
│       │   ├── values.yaml        # Shared defaults
│       │   └── values/
│       │       ├── dev.yaml       # Cluster overrides
│       │       └── prod.yaml
├── clusters/
│   ├── management/
│   │   └── cluster-values.yaml
│   ├── dev/
│   │   └── cluster-values.yaml
│   └── prod/
│       └── cluster-values.yaml
├── apps/
│   └── README.md                  # Placeholder for tenant workloads
└── scripts/
    ├── validate.py
    ├── cluster.py
    └── postgres.py
```

## Dependency and sync wave ordering (gitops)

ArgoCD sync waves control install order within a cluster:

| Wave | Components | Reason |
|------|-----------|--------|
| 0 | ArgoCD agent (on workload clusters) | Must exist before anything else |
| 1 | cert-manager, external-secrets, envoy-gateway | Core controllers, no dependencies |
| 2 | external-dns, configuration (ESO stores, certs, gateway) | Depends on wave 1 |
| 3 | LGTM | Depends on configuration (certs, secrets) |
| 4 | opentelemetry | Depends on LGTM backends |
| 5 | clickhouse-operator | Depends on cert-manager |
| 6 | clickhouse, postgres-operator | Depends on clickhouse-operator, cert-manager |
| 7 | langfuse, postgres | Depends on clickhouse, postgres-operator |
| 8 | sonarqube | Depends on postgres |
