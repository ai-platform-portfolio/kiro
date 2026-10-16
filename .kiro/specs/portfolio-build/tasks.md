# Portfolio Build — Tasks

## Phase 1: terraform module library

- [ ] **1.1** Scaffold `terraform` repo — directory structure, root README, `.gitignore`, `versions.tf` stub
- [ ] **1.2** `network` module — VNet, subnets, NSG per subnet, route table + associations
- [ ] **1.3** `network` module — private endpoints (for_each), private DNS zone creation + VNet links
- [ ] **1.4** `network` module — README with hub-and-spoke DNS note, inputs/outputs table
- [ ] **1.5** `aks-cluster` module — cluster, system node pool, OIDC issuer, workload identity
- [ ] **1.6** `aks-cluster` module — user node pools (for_each), explicit max_pods, README
- [ ] **1.7** `managed-identity` module — user-assigned identity, federated credentials (for_each), README
- [ ] **1.8** `key-vault` module — vault, access policies, optional private endpoint, README
- [ ] **1.9** `blob-storage` module — storage account, containers (for_each), optional private endpoint, README
- [ ] **1.10** `dns-zone` module — public/private zone, VNet links (for_each), existing zone escape hatch, README
- [ ] **1.11** `argocd-bootstrap` module — Helm release, namespace, cluster registration secrets, README
- [ ] **1.12** Root `terraform/README.md` — module catalogue, usage overview, network topology diagram

## Phase 2: bootstrap repo

- [ ] **2.1** Scaffold `bootstrap` repo — directory structure, README, `.gitignore`
- [ ] **2.2** `versions.tf` — pin azurerm ~> 4.0, helm ~> 2.0, kubernetes ~> 2.0
- [ ] **2.3** Networking — call `network` module for management, dev, prod VNets with correct ranges
- [ ] **2.4** Networking — VNet peering (management↔dev, management↔prod)
- [ ] **2.5** AKS — call `aks-cluster` module for management, dev, prod clusters
- [ ] **2.6** Identities — call `managed-identity` module for platform identity + postgres identity
- [ ] **2.7** Secrets — call `key-vault` module, configure access policies for managed identities
- [ ] **2.8** Storage — call `blob-storage` module for LGTM backends (Loki, Mimir, Tempo containers)
- [ ] **2.9** DNS — call `dns-zone` module for platform public zone + privatelink zones
- [ ] **2.10** ArgoCD — call `argocd-bootstrap` module for hub install, reference gitops repo values
- [ ] **2.11** `terraform.tfvars.example` with all required inputs documented
- [ ] **2.12** `outputs.tf` — kubeconfig outputs, ArgoCD server URL, key vault URI
- [ ] **2.13** `README.md` — full bootstrap runbook, pre-requisites, `make up` / `make down`

## Phase 3: gitops migration (Flux → ArgoCD)

- [ ] **3.1** Scaffold new `main` branch structure — bootstrap/, platform/, clusters/, apps/, scripts/
- [ ] **3.2** Hub ArgoCD values — `bootstrap/management/argocd-values.yaml`
- [ ] **3.3** Self-manage Application — `bootstrap/management/self-manage.yaml`
- [ ] **3.4** Cluster values files — management, dev, prod `cluster-values.yaml`
- [ ] **3.5** `argocd-agents` ApplicationSet — Argo-on-Argo, deploys agents to workload clusters
- [ ] **3.6** `cert-manager` ApplicationSet + package values
- [ ] **3.7** `external-secrets` ApplicationSet + package values (ESO + AKV store config)
- [ ] **3.8** `envoy-gateway` ApplicationSet + package values (gateway config per cluster)
- [ ] **3.9** `external-dns` ApplicationSet + package values
- [ ] **3.10** `lgtm` ApplicationSet + package values (Loki, Grafana, Tempo, Mimir per cluster)
- [ ] **3.11** `opentelemetry` ApplicationSet + package values
- [ ] **3.12** `clickhouse` ApplicationSet + package values (operator + cluster)
- [ ] **3.13** `postgres` ApplicationSet + package values (CloudNativePG + PgBouncer + Barman)
- [ ] **3.14** `langfuse` ApplicationSet + package values
- [ ] **3.15** `sonarqube` ApplicationSet + package values
- [ ] **3.16** Sync wave annotations on all ApplicationSets (see design.md wave table)
- [ ] **3.17** Validation script updates — port existing validate.py to Argo structure
- [ ] **3.18** `README.md` — full platform README, reconciliation diagram, lifecycle runbook
- [ ] **3.19** GitHub Actions workflow — validate on PR (Kustomize build, Helm lint, schema check)

## Phase 4: .github org profile

- [ ] **4.1** Update `profile/README.md` repo catalogue — add terraform, bootstrap, kiro entries
- [ ] **4.2** Add shared workflow for Linear issue check (already exists, verify it covers all repos)

## Phase 5: app monorepo (later)

- [ ] **5.1** Scaffold minimal FE (static, demonstrates platform ingress + TLS)
- [ ] **5.2** Scaffold minimal BE (demonstrates platform Postgres + observability)
- [ ] **5.3** GitHub Actions CI — build, SonarQube scan, quality gate check
- [ ] **5.4** ArgoCD Application in gitops/apps/ pointing at app repo
