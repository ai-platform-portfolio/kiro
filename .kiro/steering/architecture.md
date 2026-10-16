---
inclusion: always
---

# Architecture Decisions

## GitOps: Federated ArgoCD (not hub-and-spoke, not Flux)

**Decision:** Federated ArgoCD with a hub management cluster and independent Argo agents
on each workload cluster.

**Why not hub-and-spoke:** A single ArgoCD on a management cluster is a single point of
failure. If the management cluster is unavailable, reconciliation stops on all workload
clusters simultaneously. At Model ML scale (rapid growth, enterprise SLAs) this is
unacceptable.

**Why not Flux:** Flux reconciles from a single Git source per cluster. There is no native
mechanism to stamp the same package set across N clusters without N separate Flux instances
or N copies of manifests. ArgoCD ApplicationSets with a cluster generator solve this
natively — adding a cluster means adding one file.

**Why federated Argo:** Each workload cluster runs its own ArgoCD agent that reconciles
independently. A hub outage does not stop workload cluster reconciliation. The hub manages
the agents via ApplicationSets (Argo-on-Argo pattern). Adding a new cluster:
1. Provision cluster (Terraform)
2. Register cluster secret with hub Argo
3. Hub Argo deploys agent to new cluster via ApplicationSet
4. Agent reconciles all platform packages

This is the reconciliation-at-fleet-scale pattern that demonstrates distributed systems
depth rather than infrastructure configuration.

**Bootstrap sequence:**
```
Terraform (the only manual trigger)
  └── provisions AKS clusters
  └── helm installs ArgoCD on management cluster
  └── applies cluster registration secrets
  └── applies self-manage Application
        └── ArgoCD manages its own Helm release via Git
              └── Hub Argo deploys agents to workload clusters via ApplicationSet
                    └── Agents reconcile all platform packages
```

Terraform is the chicken-and-egg layer. Everything below it is self-healing and Git-driven.

## Networking: Hub-and-spoke (simplified for portfolio)

**Intended production design:** Hub-and-spoke with Azure Firewall DNS proxy. Private DNS
zones linked only to hub VNet. Spoke resolution via firewall DNS proxy.

**Portfolio simplification:** Direct VNet links from each private DNS zone to each spoke
VNet, in place of a DNS server. This is documented in code comments and README so reviewers
understand the intentional simplification.

VNet peering: management↔dev, management↔prod. No dev↔prod peering (correct for
hub-and-spoke isolation).

## Network ranges

```
10.0.0.0/8        overall supernet

10.1.0.0/16       management VNet (hub)
  10.1.0.0/24     system node pool
  10.1.1.0/24     user node pool
  10.1.2.0/27     private endpoints  (tightened — max 27 IPs, intentional)

10.2.0.0/16       dev VNet (spoke)
  10.2.0.0/24     system node pool
  10.2.1.0/24     user node pool
  10.2.2.0/27     private endpoints

10.3.0.0/16       prod VNet (spoke)
  10.3.0.0/24     system node pool
  10.3.1.0/24     user node pool
  10.3.2.0/27     private endpoints
```

## Terraform: opinionated modules with escape hatches

Modules provide sane defaults and require minimal inputs for the common case. Every
opinionated default is overridable via an explicit variable. The interface signals intent:
required variables are things that genuinely differ per deployment; optional variables with
defaults are things the platform team has an opinion on.

Module library lives in `terraform/`. The `bootstrap/` repo calls those modules to provision
the actual portfolio infrastructure. Modules are generic and contain no org-specific values.

## Secrets: External Secrets Operator + Azure Key Vault

Secrets never live in Git. ESO reads from Azure Key Vault via workload identity
(federated credentials, no static credentials). The pattern is established in the existing
gitops repo and carries forward.

## AI-native platform components

The platform includes components that go beyond a standard Kubernetes platform, specifically
to support AI workloads:

| Component | Role |
|-----------|------|
| Langfuse | LLM observability — traces, costs, quality of completions |
| ClickHouse | Analytical backend for Langfuse and platform analytics |
| SonarQube Developer | AI code quality gate — AI Code Assurance, taint analysis, MCP server |
| OpenTelemetry | Unified telemetry collection (traces → Tempo, metrics → Mimir, logs → Loki) |

SonarQube is kept because the portfolio also demonstrates an AI software factory pattern:
AI agents write code → SonarQube quality gate blocks AI slop → human reviews only
passing PRs. SonarQube's MCP server (2026.1+) enables fully automated inner loops where
agents self-correct against quality gate results without human intervention.

## Observability stack (LGTM)

Loki (logs) + Grafana (dashboards) + Tempo (traces) + Mimir (metrics). Individually pinned
charts. Grafana k8s-monitoring chart handles Kubernetes metrics and pod log collection via
Alloy. Applications send OTLP to the OpenTelemetry Collector.

Azure Blob Storage for Loki, Mimir, Tempo object backends (persistent across cluster
recreation). Grafana and working data on PVCs.
