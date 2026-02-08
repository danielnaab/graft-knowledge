# Grove Specifications

Living specifications for the Grove workspace management tool.

## What is Grove?

Grove is a workspace management tool for working with multiple git repositories as a unified workspace. It provides:

- Multi-repo status and navigation
- Quick capture to inbox with routing
- Cross-repo search
- Graft-aware dependency visualization
- Command execution with rich output
- Agentic integration via CLI and MCP

## Reading Guide

Grove specifications follow the [living-specifications](../../../.graft/living-specifications/) methodology. Each spec has:

- **Intent** - Why this exists, what problem it solves
- **Non-goals** - What's explicitly out of scope
- **Behavior** - Given/When/Then scenarios and edge cases
- **Constraints** - Performance, security, compatibility requirements
- **Open Questions** - Unresolved design questions
- **Decisions** - Log of decisions with rationale
- **Sources** - References and exploration notes

The specifications are **lightweight and evolving** - easier to update than the code. They focus on behavior and user experience rather than implementation details.

## Status Lifecycle

- **draft** - Initial design, not yet validated
- **working** - Being implemented, verified against code
- **stable** - Implemented and verified, trusted reference
- **deprecated** - No longer applicable

## Specifications

### Core Architecture

- [**Architecture**](./architecture.md) - System design, three-layer architecture, core concepts

### Configuration and Data

- [**Workspace Configuration**](./workspace-config.md) - workspace.yaml format, repo declarations, tags

### Features (Priority Order)

Priority follows the [vertical slices](../../../notes/2026-02-06-grove-vertical-slices.md) implementation plan:

**Priority 1** (Slices 1-2):
- Architecture ✓
- Workspace Configuration ✓

**Priority 2** (Slice 3):
- TUI Behaviors (planned) - Layout, navigation, keybindings
- Capture (planned) - Quick capture, routing, templates

**Priority 3** (Slices 4-5):
- Repository Status (planned) - Status scripts, custom signals
- Graft Integration (planned) - Dependency display, upgrade awareness

**Later** (Slices 6-7):
- Search (planned) - Cross-repo search, indexing
- Commands (planned) - Command execution, streaming output

## Implementation Alignment

These specifications are being developed **alongside implementation** through vertical slices. Each slice implements a complete user-facing feature, and specs are written/updated to reflect validated behavior.

The relationship between notes and specs:

- **Exploration notes** ([notes/](../../../notes/)) - Design exploration, brainstorming, open-ended thinking
- **Specifications** (here) - Authoritative behavior definitions, graduated from notes when stable

## Traceability

Each specification cites its source exploration notes in the **Sources** section. Notes that graduate content to specs add "Graduated to [spec]" links.

## Agentic Integration

Grove is designed for both human and AI use:

- **CLI commands** with `--json` output for agent consumption
- **MCP server** support for real-time queries (planned)
- **Structured output** from all core operations
- **Clear separation** of engine (logic) and UI (display)

Agents can query workspace state, search across repos, and use Grove as the "workspace consciousness" between sessions.

## Related Documentation

- [Workspace UI Exploration](../../../notes/2026-02-06-workspace-ui-exploration.md) - Original architecture design
- [Grove Vertical Slices](../../../notes/2026-02-06-grove-vertical-slices.md) - Implementation roadmap
- [Grove Workflow Hub Primitives](../../../notes/2026-02-07-grove-workflow-hub-primitives.md) - Core design primitives
- [Status Check Syntax Exploration](../../../notes/2026-02-08-status-check-syntax-exploration.md) - Status script design
- [Living Specifications Guide](../../../.graft/living-specifications/docs/guides/writing-specs.md) - How to write and maintain specs
