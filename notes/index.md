---
title: "Notes Index"
status: working
---

# Notes

Working notes, brainstorming sessions, design analysis, and explorations for the Graft project.

Insights should graduate to [docs/](../docs/) when they stabilize.

## Design Analysis

- [Flat-Only Dependency Analysis (2026-01-31)](./2026-01-31-flat-only-dependency-analysis.md) - **Comprehensive exploration** of flat-only dependency model, git submodules integration, DX workflows, and existing tools survey → **Graduated to [Decision 0007](../docs/decisions/decision-0007-flat-only-dependencies.md)**
- [One-Level Dependency Exploration (2026-01-31)](./2026-01-31-one-level-dependency-exploration.md) - Initial analysis of simplified dependency model with only one level of dependencies
- [Dependency Management Exploration (2026-01-12)](./2026-01-12-dependency-management-exploration.md) - Evaluated git submodules and artifact-based composition; validated current flat layout
- [Design Improvements Analysis (2026-01-05)](./2026-01-05-design-improvements-analysis.md) - Comprehensive analysis of design recommendations; created ADRs and updated specifications

## Exploration

- [Status Check Syntax Exploration (2026-02-08)](./2026-02-08-status-check-syntax-exploration.md) - Deep exploration of status check syntax alternatives (inline scripts, declarative, expressions, helpers); concludes that simple scripts (name → path mapping) are clearest; config declares what checks exist, scripts implement how to check
- [Grove as Workflow Hub: Design Primitives (2026-02-07)](./2026-02-07-grove-workflow-hub-primitives.md) - Six simple primitives for Grove as workflow hub: custom status scripts, multiple workspaces, tags, session memory, capture routing, shell-based extensibility; explores agentic integration and "workspace map" framing
- [Grove Vertical Slices (2026-02-06)](./2026-02-06-grove-vertical-slices.md) - Seven narrow end-to-end slices for building Grove: each cuts through config → engine → TUI to deliver a demoable capability incrementally
- [Workspace UI Exploration (2026-02-06)](./2026-02-06-workspace-ui-exploration.md) - "Grove" workspace tool: TUI-first multi-repo navigation, quick capture, search, and graft awareness; three-layer Rust architecture (ratatui/gitoxide/tantivy)

## Brainstorming

- [UI Architecture Brainstorming (2026-01-07)](./2026-01-07-ui-architecture-brainstorming.md) - Browser-based UI concepts: key qualities, architectural patterns, ecosystem repositories
- [Evolution Brainstorming (2026-01-05)](./2026-01-05-evolution-brainstorming.md) - Future directions: transactions, web UI, upgrade affordances, agent philosophy

## Design Sessions

- [Upgrade Mechanisms (2026-01-01)](./2026-01-01-upgrade-mechanisms.md) - Design of change tracking, migrations, and atomic upgrades

## Implementation Planning

- [Python Implementation Plan (2026-01-03)](./2026-01-03-python-implementation-plan.md) - Implementation roadmap for the Python Graft tool

## Historical

- [Initialization (2025-12-23)](./2025-12-23-initialization.md) - Initial project setup and knowledge base creation

---

## Related

- [Docs Index](../docs/README.md) - Curated, stable documentation
- [Decisions](../docs/decisions/) - Architecture Decision Records
