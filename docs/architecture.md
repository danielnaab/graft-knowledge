---
title: "Graft Architecture"
status: draft
---

# Graft Architecture

## Overview

Graft is a task runner and git-centered package manager designed to provide:
1. Configurable task/command execution
2. Git-based dependency management
3. Reproducible development workflows

## Core Concepts

### Task Runner

Graft allows defining tasks in a configuration file (e.g., `graft.yaml`) that can be executed via CLI:

```bash
graft run <task-name>
```

Tasks can include:
- Build commands
- Test suites
- Code generation
- Deployment scripts

### Git-Centered Dependencies

Dependencies are specified using git references:
- Repository URL
- Git ref (branch, tag, commit SHA)
- Optional path within repository

This enables:
- Reproducible dependency resolution
- Version pinning via commit SHAs
- Sharing code across projects without publishing to registries

### CLI Interface (Draft)

```bash
graft resolve    # Resolve and fetch dependencies
graft run <task> # Execute a defined task
```

## Design Principles

1. **Simplicity**: Start with minimal features, add complexity only when needed
2. **Git-native**: Leverage git for versioning and distribution
3. **Reproducibility**: Lock dependencies to specific commits
4. **Composability**: Allow tasks to depend on other tasks

## Status

This architecture is in early draft stage. Core concepts are being refined based on:
- Initial scope decision (see [decision-0001-initial-scope.md](decisions/decision-0001-initial-scope.md))
- Exploration of existing tools (Make, Task, Bazel, Nix)

## Open Questions

- How should lockfiles work?
- What's the caching strategy for dependencies?
- Should tasks support parallelization?
- How to handle cross-platform compatibility?

## Sources

- [Decision 0001: Initial Scope](decisions/decision-0001-initial-scope.md)
- Meta knowledge base: [../../meta-knowledge-base/docs/meta.md](../../meta-knowledge-base/docs/meta.md)
