---
inclusion: always
---

# AI Platform Portfolio — Session Context

## What this portfolio is

A production-grade AI-native platform portfolio targeting senior platform/distributed systems
engineering roles. The design benchmark is **Model ML**: a fintech AI workflow platform that
went from stealth to $12M seed (Feb 2025) to $75M Series A (Nov 2025) — one of the largest
fintech Series A rounds ever — in under 18 months. The platform is designed to absorb that
kind of growth without re-architecting.

The portfolio demonstrates:
- Designing and operating platform infrastructure at organisational scale
- Control planes, reconciliation mechanisms, and failure handling (not just config)
- GitOps fleet management across multiple clusters
- AI-native tooling (LLM observability, AI code quality gates, agentic development patterns)
- Self-service infrastructure with strong operational defaults

The target role profile is **platform distributed systems engineering** — the distinction
between provisioning Kubernetes clusters (infrastructure config) and designing the control
planes and reconciliation systems that manage them at scale (distributed systems engineering).

## Org structure

GitHub org: `ai-platform-portfolio`

| Repo | Purpose | Status |
|------|---------|--------|
| `.github` | Org profile, repo catalogue, shared workflows | Exists |
| `gitops` | ArgoCD fleet management — the nucleus | Exists (migrating Flux → Argo) |
| `terraform` | Reusable module library | Building next |
| `bootstrap` | Calls terraform modules, provisions clusters, installs ArgoCD | Building after terraform |
| `kiro` | This repo — AI session context, architecture decisions, portfolio spec | Exists |
| `app` | Minimal FE + BE monorepo for platform-layered applications | Later |

## Non-goals

- Demonstrating software engineering (the `app` repo is intentionally minimal)
- Multi-cloud (Azure/AKS only for now; GKE exists in private infra but not surfaced)
- SRE/observability depth as the primary narrative (observability is present but supporting)
