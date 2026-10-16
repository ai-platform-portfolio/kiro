# kiro

Kiro AI session context for the `ai-platform-portfolio` org. Contains steering files that are automatically loaded into every Kiro session working on this portfolio, ensuring architectural decisions, network topology, naming conventions, and build context are consistent across sessions.

## What's in here

```
.kiro/
└── steering/
    ├── context.md       # Portfolio goal, org structure, what we're building and why
    ├── architecture.md  # All architecture decisions with rationale — GitOps, networking,
    │                    # Terraform module design, secrets, AI-native components
    └── conventions.md   # Naming conventions, module interface standards, Git workflow
```

## How to use

Clone this repo alongside the others in the portfolio workspace. Kiro picks up `.kiro/steering/` automatically and injects the context into every session. No manual setup required.

```sh
git clone https://github.com/ai-platform-portfolio/kiro
```

Add it as a folder in your VS Code / Kiro workspace alongside `gitops`, `terraform`, and `bootstrap`.

## Why this exists

AI sessions don't carry memory across conversations. Without explicit context, every session starts cold — re-establishing what the architecture is, why decisions were made, and what the current state is. These steering files solve that.

They also serve as a living architecture decision record (ADR) for the portfolio itself, useful for anyone reviewing the org to understand the intent behind the design.
