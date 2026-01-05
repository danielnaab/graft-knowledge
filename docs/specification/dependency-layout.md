---
title: "Dependency Layout Specification"
date: 2026-01-05
status: draft
version: v2
---

# Dependency Layout Specification

## Overview

This specification defines how Graft organizes dependencies on disk to maximize ergonomics for knowledge base consumption while avoiding common package manager pitfalls.

**Design Goals:**
1. Enable ergonomic markdown linking between repositories
2. Support recursive dependencies (grafts depending on grafts)
3. Avoid path length issues, duplication, and naming conflicts
4. Provide predictable, short paths for human use
5. Enable efficient storage and reproducible builds

---

## Consumption Patterns Analysis

### How Dependencies Are Consumed

Knowledge base dependencies differ from code dependencies in critical ways:

**1. Direct Reference** - Markdown links to dependency content
```markdown
[Architecture Patterns](../.graft/meta-kb/docs/architecture.md)
![Diagram](../.graft/meta-kb/assets/flow.svg)
```

**2. Transclusion** - Including content from dependencies
```markdown
<!-- Include shared template -->
{{include ../.graft/templates/header.md}}
```

**3. Asset Usage** - Images, diagrams, files referenced in builds
```bash
# Copy shared assets
cp .graft/brand-kb/logos/*.svg public/
```

**4. Search and Indexing** - Tools scanning dependency content
```bash
# Index all knowledge
grep -r "concept" . .graft/*/docs/
```

**5. Script Execution** - Running tools from dependencies
```bash
# Use dep's validation script
.graft/standards-kb/scripts/validate.sh
```

**6. Recursive Reference** - Dependencies referencing their own dependencies
```
your-project depends on meta-kb
  meta-kb depends on standards-kb
    standards-kb depends on templates-kb
```

### Critical Requirements

From these patterns, we derive:

- **Short paths**: Human-friendly linking `../.graft/dep-name/`
- **Predictable locations**: Always know where a dep lives
- **Deduplication**: Don't clone same dep+version multiple times
- **Isolation**: Each dep can reference its own dependencies
- **Conflict detection**: Explicitly fail on version conflicts
- **No magic**: Transparent, inspectable structure

---

## Package Manager Pitfalls

### What Other Systems Got Wrong

**npm pre-v3: Nested node_modules**
```
node_modules/
  a/
    node_modules/
      b/
        node_modules/
          c/  # Path length issues on Windows
```
- ❌ Extreme duplication
- ❌ Path length limits
- ❌ Inefficient disk usage

**npm v3+: Hoisting and flattening**
```
node_modules/
  a/
  b/  # Hoisted from a's deps
  c/  # Hoisted from b's deps
```
- ❌ Non-deterministic structure
- ❌ Phantom dependencies (using undeclared deps)
- ❌ Complex resolution algorithm

**Python pip: Global namespace**
- ❌ Only one version per environment
- ❌ Dependency conflicts hard to resolve
- ❌ No isolation between projects

**Go early GOPATH: Rigid structure**
- ❌ Requires specific directory layout
- ❌ Global workspace, no isolation

### What They Got Right

✅ **Lockfiles** (npm, yarn, poetry, bundler) - Reproducibility
✅ **Semantic versioning** - Clear expectations
✅ **Deduplication where possible** - Efficiency
✅ **Explicit resolution** (Go modules) - Predictability
✅ **Content-addressed storage** (pnpm) - Efficiency

---

## Recursive Dependency Resolution Strategies

### Strategy Comparison

#### 1. Nested Layout (npm pre-v3)

```
.graft/
  meta-kb/
    .graft/
      standards-kb/
        .graft/
          templates-kb/
```

**Pros:**
- ✅ Complete isolation
- ✅ Simple mental model
- ✅ No conflicts possible

**Cons:**
- ❌ Massive duplication if shared deps
- ❌ Deep paths (ergonomics suffer)
- ❌ Inefficient disk usage
- ❌ Can't reference peer dependencies

**Verdict:** ❌ Too many downsides for knowledge bases

---

#### 2. Flat with Hoisting (npm v3+)

```
.graft/
  meta-kb/          # Your dep
  standards-kb/     # Hoisted from meta-kb
  templates-kb/     # Hoisted from standards-kb
```

**Pros:**
- ✅ Efficient (no duplication)
- ✅ Short paths
- ✅ Works well when deps align

**Cons:**
- ❌ Non-deterministic structure
- ❌ Phantom dependencies (can use undeclared deps)
- ❌ Complex resolution logic
- ❌ Hard to understand which dep depends on what

**Verdict:** ❌ Too much implicit behavior

---

#### 3. Flat with Namespacing (Go modules style)

```
.graft/
  github.com/org/meta-kb/
  github.com/org/standards-kb/
  github.com/other/templates-kb/
```

**Pros:**
- ✅ No name collisions
- ✅ Predictable structure
- ✅ Explicit source

**Cons:**
- ❌ Long paths (ergonomics suffer)
- ❌ URL in path is awkward for linking
- ❌ Different repos can have same name

**Verdict:** ❌ Paths too long for ergonomics

---

#### 4. Flat with Explicit Resolution (Recommended)

```
project/
├── graft.yaml       # Direct dependencies
├── graft.lock       # Locked versions
├── graft-tree.json  # Full dependency tree metadata
└── .graft/
    ├── meta-kb/         # Direct dep
    ├── standards-kb/    # meta-kb's dep (flattened)
    └── templates-kb/    # standards-kb's dep (flattened)
```

**Pros:**
- ✅ Short, predictable paths: `.graft/<name>/`
- ✅ Efficient (deduplication)
- ✅ Explicit (metadata shows tree)
- ✅ No phantom dependencies
- ✅ Tooling can validate and visualize
- ✅ Conflict detection

**Cons:**
- ⚠️ Requires dependency resolution algorithm
- ⚠️ Must detect and fail on version conflicts

**Verdict:** ✅ Best balance for knowledge bases

---

#### 5. Content-Addressed with Virtual Trees (pnpm style)

```
.graft/
  .store/
    meta-kb@abc123/
    standards-kb@def456/
  meta-kb -> .store/meta-kb@abc123/
  standards-kb -> .store/standards-kb@def456/
```

**Pros:**
- ✅ Maximum deduplication
- ✅ Efficient for monorepos

**Cons:**
- ❌ Symlinks can be fragile
- ❌ More complex
- ❌ Overkill for typical use cases

**Verdict:** ⚠️ Possibly future optimization

---

## Recommended Design: Flat Layout with Dependency Tree

### Core Design

**Directory Structure:**
```
project/
├── graft.yaml          # User-edited: direct dependencies
├── graft.lock          # Auto-generated: locked versions
├── graft-tree.json     # Auto-generated: full dependency tree
└── .graft/
    ├── meta-kb/        # Dependency: cloned once, used everywhere
    ├── standards-kb/   # Transitive dep from meta-kb
    └── templates-kb/   # Transitive dep from standards-kb
```

**Key Principle:** Remove `/deps/` subdirectory - it's superfluous. Place dependencies directly in `.graft/`.

### File Formats

**graft.yaml** (unchanged)
```yaml
apiVersion: graft/v0
deps:
  meta-kb: "https://github.com/org/meta.git#v2.0.0"
```

**graft.lock** (unchanged)
```yaml
apiVersion: graft/v0
dependencies:
  meta-kb:
    source: "https://github.com/org/meta.git"
    ref: "v2.0.0"
    commit: "abc123..."
    consumed_at: "2026-01-05T10:30:00Z"
```

**graft-tree.json** (NEW - full resolved tree)
```json
{
  "version": "graft/v0",
  "generated_at": "2026-01-05T10:30:00Z",
  "tree": {
    "meta-kb": {
      "path": ".graft/meta-kb",
      "source": "https://github.com/org/meta.git",
      "ref": "v2.0.0",
      "commit": "abc123...",
      "direct": true,
      "dependencies": {
        "standards-kb": {
          "path": ".graft/standards-kb",
          "source": "https://github.com/org/standards.git",
          "ref": "v1.5.0",
          "commit": "def456...",
          "direct": false,
          "dependencies": {
            "templates-kb": {
              "path": ".graft/templates-kb",
              "source": "https://github.com/org/templates.git",
              "ref": "v1.0.0",
              "commit": "ghi789...",
              "direct": false,
              "dependencies": {}
            }
          }
        }
      }
    }
  },
  "flat_map": {
    "meta-kb": ".graft/meta-kb",
    "standards-kb": ".graft/standards-kb",
    "templates-kb": ".graft/templates-kb"
  }
}
```

### Dependency Resolution Algorithm

```python
def resolve_dependencies(
    direct_deps: dict[str, str],
    existing_resolutions: dict[str, LockEntry] = {}
) -> dict[str, LockEntry]:
    """
    Resolve all dependencies recursively with conflict detection.

    Returns flat map of name -> LockEntry with full tree metadata.
    Raises ConflictError if incompatible versions required.
    """
    resolved = {}
    queue = [(name, url, is_direct=True) for name, url in direct_deps.items()]

    while queue:
        name, url, is_direct = queue.pop(0)

        # Skip if already resolved with same version
        if name in resolved:
            existing = resolved[name]
            if existing.source == url and existing.ref == parse_ref(url):
                continue
            else:
                # CONFLICT: same name, different version
                raise ConflictError(
                    f"Dependency conflict: {name}\n"
                    f"  Required: {url}\n"
                    f"  Existing: {existing.source}#{existing.ref}"
                )

        # Clone/fetch dependency
        dep_path = f".graft/{name}"
        clone_or_fetch(url, dep_path)

        # Read its graft.yaml to find transitive deps
        dep_config = parse_config(f"{dep_path}/graft.yaml")

        # Record resolution
        resolved[name] = LockEntry(
            source=parse_source(url),
            ref=parse_ref(url),
            commit=get_commit_sha(dep_path),
            consumed_at=now(),
            direct=is_direct,
            dependencies=dep_config.deps
        )

        # Add transitive deps to queue
        for trans_name, trans_url in dep_config.deps.items():
            queue.append((trans_name, trans_url, is_direct=False))

    return resolved
```

### Conflict Handling

**Scenario: Version Conflict**
```
Project depends on:
  - meta-kb v2.0.0 (requires standards-kb v1.5.0)
  - docs-kb v1.0.0 (requires standards-kb v1.0.0)

Result: CONFLICT
```

**Resolution Strategy:**
1. **Detect conflict** during resolution
2. **Fail explicitly** with clear error message
3. **User must resolve** by:
   - Upgrading meta-kb or docs-kb to compatible versions
   - Using version ranges if supported (future)
   - Forking and patching one dependency

**Why explicit failure?**
- Knowledge bases are human-consumed
- Version conflicts often mean incompatible content structures
- Better to fail than have inconsistent content
- Encourages ecosystem coordination

### Migration Path

**Phase 1: Support both layouts**
- Accept `.graft/deps/` (current)
- Accept `.graft/` (new)
- Generate graft-tree.json on resolve

**Phase 2: Migrate existing projects**
- Add `graft migrate-layout` command
- Moves `.graft/deps/*` to `.graft/*`
- Generates graft-tree.json

**Phase 3: Deprecate old layout**
- Warn on `.graft/deps/` usage
- Update docs to show new layout

**Phase 4: Remove old layout support**
- Only support `.graft/<name>/`

---

## Benefits

### For Users

**Ergonomic linking:**
```markdown
<!-- Before: Long path -->
[Pattern](../.graft/deps/meta-kb/docs/pattern.md)

<!-- After: Shorter -->
[Pattern](../.graft/meta-kb/docs/pattern.md)
```

**Predictable structure:**
```bash
# Always know where deps are
ls .graft/
# meta-kb  standards-kb  templates-kb

# Tree structure is explicit
cat graft-tree.json
```

**No surprises:**
- Can't accidentally use undeclared dependencies
- Conflicts fail loudly and early
- Clear error messages guide resolution

### For Tool Builders

**Validation:**
```bash
# Check all deps match lock
graft validate --check-tree

# Show full dependency tree
graft tree --show-all
```

**Visualization:**
```bash
# Graph the dependency tree
graft tree --graph | dot -Tpng > deps.png
```

**Analysis:**
```bash
# Find unused dependencies
graft analyze --unused

# Check for dependency cycles
graft analyze --cycles
```

---

## Open Questions

### 1. Version Ranges?

Should we support version ranges in graft.yaml?

```yaml
deps:
  meta-kb: "https://github.com/org/meta.git#^v2.0.0"
```

**Pros:** More flexibility, easier to avoid conflicts
**Cons:** Less reproducible, complexity
**Decision:** Start without, add if needed

### 2. Peer Dependencies?

Should deps declare what they expect consumer to have?

```yaml
deps:
  standards-kb: "..."

peer_deps:
  templates-kb: "v1.0.0"  # Must be provided by consumer
```

**Pros:** Solves some conflict scenarios
**Cons:** Adds complexity
**Decision:** Defer until we see the need

### 3. Workspace Support?

Support monorepos with multiple projects sharing deps?

```
workspace/
  project-a/
    graft.yaml
  project-b/
    graft.yaml
  .graft/  # Shared deps
```

**Pros:** Efficiency for monorepos
**Cons:** Complexity, scope creep
**Decision:** Future enhancement

---

## Implementation Plan

### Phase 1: Core Resolution (Week 1)
- [ ] Implement recursive dependency resolution algorithm
- [ ] Generate graft-tree.json during `graft resolve`
- [ ] Support both `.graft/deps/` and `.graft/` layouts
- [ ] Add conflict detection and clear error messages

### Phase 2: Migration Support (Week 2)
- [ ] Add `graft migrate-layout` command
- [ ] Update all documentation
- [ ] Add warnings for old layout
- [ ] Create migration guide

### Phase 3: Tree Tooling (Week 3)
- [ ] Add `graft tree` command to visualize dependencies
- [ ] Add `graft validate --check-tree`
- [ ] Add `graft analyze` for unused/cycle detection
- [ ] Update lock file format to include transitive deps

### Phase 4: Optimization (Week 4)
- [ ] Consider content-addressed storage for efficiency
- [ ] Add parallel cloning for speed
- [ ] Optimize for large dependency graphs
- [ ] Performance testing and benchmarks

---

## References

- **Similar systems:**
  - pnpm (content-addressed storage)
  - Go modules (explicit minimal version selection)
  - Bazel (explicit dependency graph)

- **Related specifications:**
  - [graft.yaml Format](./graft-yaml-format.md)
  - [Lock File Format](./lock-file-format.md)
  - [Core Operations](./core-operations.md)

- **Discussions:**
  - User feedback on ergonomics
  - Performance requirements
  - Conflict resolution strategies

---

## Appendix: Example Scenarios

### Scenario 1: Simple Project

**Setup:**
```yaml
# graft.yaml
deps:
  meta-kb: "https://github.com/org/meta.git#v2.0.0"
```

**After `graft resolve`:**
```
project/
├── graft.yaml
├── graft.lock
├── graft-tree.json
└── .graft/
    ├── meta-kb/
    ├── standards-kb/  (from meta-kb)
    └── templates-kb/  (from standards-kb)
```

**Linking:**
```markdown
[Concept](../.graft/meta-kb/docs/concept.md)
```

### Scenario 2: Conflict Detection

**Setup:**
```yaml
# graft.yaml
deps:
  meta-kb: "https://github.com/org/meta.git#v2.0.0"
  docs-kb: "https://github.com/org/docs.git#v1.0.0"
```

Where:
- meta-kb v2.0.0 requires standards-kb v1.5.0
- docs-kb v1.0.0 requires standards-kb v1.0.0

**Result:**
```
Error: Dependency conflict detected

Dependency: standards-kb
  Required by meta-kb: v1.5.0
  Required by docs-kb: v1.0.0

These versions are incompatible. Please:
1. Check if newer versions align
2. Contact maintainers about compatibility
3. Use only one of the conflicting dependencies
```

### Scenario 3: Shared Dependency

**Setup:**
```yaml
# graft.yaml
deps:
  meta-kb: "https://github.com/org/meta.git#v2.0.0"
  docs-kb: "https://github.com/org/docs.git#v2.0.0"
```

Where both require templates-kb v1.0.0

**Result:**
```
project/
└── .graft/
    ├── meta-kb/
    ├── docs-kb/
    ├── standards-kb/  (from meta-kb)
    └── templates-kb/  (shared - cloned once)
```

**graft-tree.json shows sharing:**
```json
{
  "tree": {
    "meta-kb": {
      "dependencies": {
        "templates-kb": { "path": ".graft/templates-kb", "shared": true }
      }
    },
    "docs-kb": {
      "dependencies": {
        "templates-kb": { "path": ".graft/templates-kb", "shared": true }
      }
    }
  }
}
```

---

## Changelog

- **2026-01-05**: Initial draft (v2 design)
  - Analyzed consumption patterns
  - Evaluated recursive dependency strategies
  - Proposed flat layout with explicit resolution
  - Defined graft-tree.json format
  - Planned migration path
