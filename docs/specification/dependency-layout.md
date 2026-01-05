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
├── graft.lock       # Extended: all resolved deps (direct + transitive)
└── .graft/
    ├── meta-kb/         # Direct dep
    ├── standards-kb/    # meta-kb's dep (flattened)
    └── templates-kb/    # standards-kb's dep (flattened)
```

**Pros:**
- ✅ Short, predictable paths: `.graft/<name>/`
- ✅ Efficient (deduplication)
- ✅ Explicit (lock file shows all consumed deps)
- ✅ No phantom dependencies
- ✅ Tooling can validate and visualize
- ✅ Conflict detection
- ✅ Single source of truth (no separate tree file)

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

## Recommended Design: Flat Layout with Extended Lock File

### Core Design

**Directory Structure:**
```
project/
├── graft.yaml          # User-edited: direct dependencies
├── graft.lock          # Auto-generated: ALL resolved deps (direct + transitive)
└── .graft/
    ├── meta-kb/        # Dependency: cloned once, used everywhere
    ├── standards-kb/   # Transitive dep from meta-kb
    └── templates-kb/   # Transitive dep from standards-kb
```

**Key Principles:**
1. Remove `/deps/` subdirectory - it's superfluous. Place dependencies directly in `.graft/`.
2. Lock file contains ALL resolved dependencies (not just direct ones)
3. Single source of truth - no separate tree file needed

### File Formats

**graft.yaml** (unchanged)
```yaml
apiVersion: graft/v0
deps:
  meta-kb: "https://github.com/org/meta.git#v2.0.0"
```

**graft.lock** (EXTENDED - includes all resolved dependencies)
```yaml
apiVersion: graft/v0

# All resolved dependencies (direct + transitive)
# Ordered for readability (direct first, then transitive alphabetically)
dependencies:
  meta-kb:
    source: "https://github.com/org/meta.git"
    ref: "v2.0.0"
    commit: "abc123..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: true              # NEW: Is this a direct dependency?
    requires: ["standards-kb"] # NEW: What this dep needs (list of dep names)

  standards-kb:
    source: "https://github.com/org/standards.git"
    ref: "v1.5.0"
    commit: "def456..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: false
    required_by: ["meta-kb"]  # NEW: Which deps need this (for auditing)
    requires: ["templates-kb"]

  templates-kb:
    source: "https://github.com/org/templates.git"
    ref: "v1.0.0"
    commit: "ghi789..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: false
    required_by: ["standards-kb"]
    requires: []               # Leaf dependency
```

**Why this format?**

1. **Single source of truth**: All consumed versions in one place
2. **Follows conventions**: npm, yarn, poetry all include transitive deps in lock files
3. **Enables features**:
   - Upgrade preview: "Upgrading meta-kb will also update 2 transitive deps"
   - Garbage collection: "templates-kb is no longer needed, remove from .graft/"
   - Security auditing: "Check all consumed deps for vulnerabilities"
   - Dependency visualization: Can reconstruct tree from requires/required_by
4. **Reproducible**: Pins exact versions of ALL dependencies, not just direct
5. **No redundant files**: Don't need separate tree metadata

### Dependency Resolution Algorithm

```python
@dataclass
class ResolvedDep:
    source: str
    ref: str
    commit: str
    consumed_at: str
    direct: bool
    requires: list[str]
    required_by: list[str]

def resolve_dependencies(
    direct_deps: dict[str, str]
) -> dict[str, ResolvedDep]:
    """
    Resolve all dependencies recursively with conflict detection.

    Returns flat map of name -> ResolvedDep suitable for graft.lock.
    Raises ConflictError if incompatible versions required.
    """
    resolved: dict[str, ResolvedDep] = {}
    queue: list[tuple[str, str, bool, str | None]] = []

    # Initialize queue with direct dependencies
    for name, url in direct_deps.items():
        queue.append((name, url, True, None))  # (name, url, is_direct, parent)

    while queue:
        name, url, is_direct, parent = queue.pop(0)

        source = parse_source(url)  # Strip #ref
        ref = parse_ref(url)        # Extract ref

        # Check if already resolved
        if name in resolved:
            existing = resolved[name]
            if existing.source == source and existing.ref == ref:
                # Same dep, same version - just update required_by
                if parent and parent not in existing.required_by:
                    existing.required_by.append(parent)
                continue
            else:
                # CONFLICT: same name, different version
                raise ConflictError(
                    f"Dependency conflict: {name}\n"
                    f"  Required by {parent}: {source}#{ref}\n"
                    f"  Already resolved: {existing.source}#{existing.ref}\n"
                    f"  Required by: {', '.join(existing.required_by)}"
                )

        # Clone/fetch dependency to .graft/<name>/
        dep_path = f".graft/{name}"
        clone_or_fetch(url, dep_path)

        # Read its graft.yaml to find transitive dependencies
        dep_config = parse_config(f"{dep_path}/graft.yaml")
        transitive_deps = dep_config.deps or {}

        # Record resolution
        resolved[name] = ResolvedDep(
            source=source,
            ref=ref,
            commit=get_commit_sha(dep_path),
            consumed_at=now(),
            direct=is_direct,
            requires=list(transitive_deps.keys()),
            required_by=[parent] if parent else []
        )

        # Add transitive deps to queue
        for trans_name, trans_url in transitive_deps.items():
            queue.append((trans_name, trans_url, False, name))

    return resolved

def write_lock_file(resolved: dict[str, ResolvedDep]) -> None:
    """Write extended graft.lock with all resolved dependencies."""
    lock_data = {
        "apiVersion": "graft/v0",
        "dependencies": {}
    }

    # Order: direct deps first, then transitive (alphabetically)
    direct = {k: v for k, v in resolved.items() if v.direct}
    transitive = {k: v for k, v in resolved.items() if not v.direct}

    for deps in [direct, transitive]:
        for name, dep in sorted(deps.items()):
            lock_data["dependencies"][name] = {
                "source": dep.source,
                "ref": dep.ref,
                "commit": dep.commit,
                "consumed_at": dep.consumed_at,
                "direct": dep.direct,
                "requires": dep.requires,
                "required_by": dep.required_by
            }

    write_yaml("graft.lock", lock_data)
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

# All consumed versions in lock file
cat graft.lock
# Shows: meta-kb (direct), standards-kb (transitive), templates-kb (transitive)
```

**No surprises:**
- Can't accidentally use undeclared dependencies
- Conflicts fail loudly and early
- Clear error messages guide resolution

### For Tool Builders

**Validation:**
```bash
# Check all deps match lock file
graft validate --check-deps

# Show full dependency tree (reads from graft.lock)
graft tree --show-all
```

**Visualization:**
```bash
# Graph the dependency tree (from graft.lock requires/required_by)
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
- [ ] Extend graft.lock format to include all resolved deps (direct + transitive)
- [ ] Add `direct`, `requires`, `required_by` fields to lock entries
- [ ] Support both `.graft/deps/` and `.graft/` layouts
- [ ] Add conflict detection and clear error messages

### Phase 2: Migration Support (Week 2)
- [ ] Add `graft migrate-layout` command to move deps from /deps/ to root
- [ ] Update lock file reader/writer for new format (backward compatible)
- [ ] Update all documentation
- [ ] Add warnings for old layout
- [ ] Create migration guide

### Phase 3: Tree Tooling (Week 3)
- [ ] Add `graft tree` command to visualize dependencies (reads from graft.lock)
- [ ] Add `graft validate --check-deps` to verify lock matches .graft/
- [ ] Add `graft analyze --unused` to find deps no longer needed
- [ ] Add `graft analyze --cycles` to detect circular dependencies

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
├── graft.lock       # Extended: includes all 3 deps
└── .graft/
    ├── meta-kb/
    ├── standards-kb/  (from meta-kb)
    └── templates-kb/  (from standards-kb)
```

**graft.lock contents:**
```yaml
apiVersion: graft/v0
dependencies:
  meta-kb:
    source: "https://github.com/org/meta.git"
    ref: "v2.0.0"
    commit: "abc123..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: true
    requires: ["standards-kb"]
    required_by: []

  standards-kb:
    source: "https://github.com/org/standards.git"
    ref: "v1.5.0"
    commit: "def456..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: false
    requires: ["templates-kb"]
    required_by: ["meta-kb"]

  templates-kb:
    source: "https://github.com/org/templates.git"
    ref: "v1.0.0"
    commit: "ghi789..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: false
    requires: []
    required_by: ["standards-kb"]
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

**graft.lock shows sharing via required_by:**
```yaml
apiVersion: graft/v0
dependencies:
  meta-kb:
    direct: true
    requires: ["standards-kb", "templates-kb"]
    required_by: []

  docs-kb:
    direct: true
    requires: ["templates-kb"]
    required_by: []

  standards-kb:
    direct: false
    requires: []
    required_by: ["meta-kb"]

  templates-kb:
    direct: false
    requires: []
    required_by: ["meta-kb", "docs-kb"]  # Shared by both!
```

**Key insight:** When `templates-kb` appears in multiple `required_by` lists, it's a shared dependency cloned only once.

---

## Changelog

- **2026-01-05 (v2.1)**: Simplified to use extended graft.lock
  - Removed graft-tree.json (redundant metadata file)
  - Extended graft.lock to include all resolved dependencies
  - Added fields: `direct`, `requires`, `required_by`
  - Follows npm/yarn/poetry conventions (transitive deps in lock)
  - Updated all examples and algorithm

- **2026-01-05 (v2.0)**: Initial draft
  - Analyzed consumption patterns
  - Evaluated recursive dependency strategies
  - Proposed flat layout with explicit resolution
  - Planned migration path
