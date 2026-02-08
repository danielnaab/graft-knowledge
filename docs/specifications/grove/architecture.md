---
status: draft
last-verified: 2026-02-08
owners: [human, agent]
---

# Grove Architecture

## Intent

Define Grove's high-level system architecture to guide implementation and ensure clear separation of concerns. Grove is a workspace management tool that helps developers work with multiple git repositories as a unified workspace, with graft-awareness as a feature layer.

## Non-goals

- **Not a graft dashboard** - Grove is a developer workspace tool that happens to understand graft, not primarily a graft UI
- **Not deeply coupled to graft** - Grove reads graft's data files (graft.yaml, graft.lock) but doesn't depend on graft as a library
- **Not a replacement for editors/terminals** - Grove is a "departure board" showing where to go, not the destination itself
- **Not a daemon or server** - Grove is a library consumed by UIs (TUI, native, web), not a background service

## Behavior

### Three-Layer Architecture

Grove follows a clean three-layer architecture:

```
┌─────────────────────────────────────────────────────────┐
│  Layer 3: User Interfaces                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │   TUI    │  │  macOS   │  │   Web    │               │
│  │(ratatui) │  │ (SwiftUI)│  │          │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       └──────────────┴─────────────┘                     │
│                      │                                    │
├──────────────────────┼────────────────────────────────────┤
│  Layer 2: Workspace Engine                                │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Multi-repo registry · Search index · File watcher │   │
│  │  Capture routing · Command dispatch · Git ops      │   │
│  │  Graft metadata reader                             │   │
│  └────────────────────────┬───────────────────────────┘   │
│                           │                               │
├───────────────────────────┼───────────────────────────────┤
│  Layer 1: Foundation Libraries                            │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐                 │
│  │gitoxide │  │ tantivy  │  │ notify   │                 │
│  │ (git)   │  │ (search) │  │(fs watch)│                 │
│  └─────────┘  └──────────┘  └──────────┘                 │
└───────────────────────────────────────────────────────────┘
```

**Layer 1: Foundation Libraries** - Existing Rust crates for git, search, and file watching:
- **gitoxide** - Pure Rust git implementation (protocol v2, no C dependencies)
- **tantivy** - Lucene-class full-text search engine (embeddable)
- **notify** - Cross-platform file system event watching

**Layer 2: Workspace Engine** - The core logic we build:
- Multi-repo registry (tracks repositories in workspace)
- Search index (maintains cross-repo search capability)
- File watcher (detects changes across workspace)
- Capture routing (directs quick captures to correct locations)
- Command dispatch (executes commands in repo contexts)
- Git operations (status, commit, push via gitoxide)
- Graft metadata reader (parses graft.yaml/graft.lock when present)

**Layer 3: User Interfaces** - Interchangeable UIs consuming the engine:
- **TUI** (ratatui) - Terminal interface, primary focus
- **macOS/iOS** (SwiftUI via UniFFI) - Native Apple apps, future
- **Web** (HTTP/WebSocket) - Browser-based interface, future

### Workspace as Core Primitive

```gherkin
Given a workspace configuration file at ~/.config/grove/workspace.yaml
When Grove launches
Then it loads all configured repositories into the workspace registry
And displays their aggregate status
```

A **workspace** is a named collection of git repositories that you work with together:

```yaml
name: "my-project"
repositories:
  - path: ~/src/graft-knowledge
  - path: ~/src/meta-knowledge-base
  - path: ~/src/my-app

capture:
  inbox: ~/src/graft-knowledge/notes/inbox/
  auto_commit: true
```

The workspace config is minimal - it declares which repos are in scope and basic settings. Everything else is derived from the repos themselves.

### Graft Awareness as Feature Layer

```gherkin
Given a repository in the workspace has graft.yaml
When viewing that repository's details
Then Grove displays its graft dependencies and their status
```

```gherkin
Given a repository without graft.yaml
When viewing that repository's details
Then Grove shows git status and files (no graft information)
```

Grove doesn't require graft - it's useful even for plain git repos. When graft.yaml exists, Grove:
1. Parses the file to understand dependencies and commands
2. Reads graft.lock to show dependency versions
3. Can shell out to `graft` CLI for mutations (upgrade, apply)

This is **loose coupling** - Grove reads graft's data files but doesn't depend on graft as a library.

### Engine-UI Separation

```gherkin
Given the workspace engine exposes its operations as a library
When building a new UI (TUI, native, web)
Then it can consume the engine without reimplementing logic
```

The workspace engine is a **library**, not a daemon or API server. UIs link against it:

- **TUI**: Direct Rust library linkage
- **macOS/iOS**: Via UniFFI-generated Swift bindings
- **Web**: Thin HTTP/WebSocket wrapper around engine

All business logic lives in the engine. UIs only handle display and user interaction.

### CLI and Agentic Integration

```gherkin
Given Grove provides a CLI alongside the TUI
When an agent or script needs workspace information
Then it can call `grove status --json` to get structured output
```

```gherkin
Given an agent wants to capture a thought
When it runs `grove capture "@repo-name message"`
Then the capture is routed and committed like any UI capture
```

Every core operation has both a TUI interface and a CLI/machine-readable interface. This enables:
- Scripting and automation
- Agent integration (Claude Code, Cursor, etc.)
- MCP server implementation (future)
- Testing without UI

## Constraints

### Performance
- Workspace load time < 500ms for 10 repos
- Search results < 100ms for indexed search
- Status check updates < 50ms per repo
- TUI render at 60fps minimum

### Security
- No network access (local-only tool)
- Respect git credential helpers for operations
- Don't expose sensitive data in JSON output without flags

### Compatibility
- Works with any git repository (graft optional)
- Supports git 2.25+ features (submodules, sparse checkout)
- Cross-platform (Linux, macOS, Windows)
- Terminal compatibility via ratatui abstractions

### Distribution
- Single binary with no external dependencies
- Self-contained (embedded search index, no external DB)
- No installation ceremony (download and run)

## Open Questions

- [ ] Should the engine maintain state between invocations (daemon) or rebuild on launch?
- [ ] Should search indexing be incremental (file watcher) or full rebuild?
- [ ] Should UniFFI be used for native apps or direct Swift/Kotlin bindings?
- [ ] Should MCP server support be built-in or a separate wrapper?
- [ ] How should the engine handle concurrent operations (multiple UIs)?

## Decisions

- **2026-02-06**: Chose Rust for implementation
  - Strong ecosystem for all layers (ratatui, gitoxide, tantivy)
  - UniFFI path to native Apple apps
  - Single binary distribution
  - Performance characteristics fit workspace tool use case

- **2026-02-06**: Engine as library, not daemon
  - Simpler architecture (no IPC, no state sync)
  - Easier testing and debugging
  - Can always add daemon wrapper later if needed
  - Fits "departure board" mental model (launch, check, go elsewhere)

- **2026-02-06**: Graft awareness via file reading, not library coupling
  - Keeps Grove useful without graft
  - graft.yaml/graft.lock are simple YAML (no complex parser needed)
  - Can delegate mutations to `graft` CLI when it exists
  - Clean separation of concerns

- **2026-02-07**: CLI-first for every feature
  - TUI and CLI share the same engine
  - Enables agent integration from day one
  - Makes testing easier (no UI needed)
  - Structured output (`--json`) for machine consumption

## Sources

- [Workspace UI Exploration (2026-02-06)](../../../notes/2026-02-06-workspace-ui-exploration.md) - Original architecture design, three-layer model, workspace concept
- [Grove Workflow Hub Primitives (2026-02-07)](../../../notes/2026-02-07-grove-workflow-hub-primitives.md) - CLI integration, agentic workflows, engine-UI separation
- [ratatui](https://ratatui.rs/) - Rust TUI framework
- [gitoxide](https://github.com/GitoxideLabs/gitoxide) - Pure Rust git
- [tantivy](https://github.com/quickwit-oss/tantivy) - Rust search engine
- [UniFFI](https://mozilla.github.io/uniffi-rs/) - Rust↔Swift/Kotlin bindings
