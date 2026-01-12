---
title: "Work Log: Dependency Management Architecture Exploration"
date: 2026-01-12
status: completed
authors: ["Architecture Review"]
---

# Work Log: Dependency Management Architecture Exploration

## Objective

Reevaluate Graft's dependency management architecture by exploring two fundamental questions:
1. Should Graft use git submodules instead of direct checkouts to `.graft/`?
2. Should Graft clone transitive dependencies, or should upstream grafts create self-contained artifacts?

This exploration validates the current architectural decisions against alternative approaches and confirms alignment with Graft's design philosophy.

## Background

Graft currently uses a **flat directory layout** with direct git checkouts:
```
project/
├── graft.yaml          # Direct dependencies
├── graft.lock          # Complete resolved graph (direct + transitive)
└── .graft/
    ├── meta-kb/        # Direct dependency
    ├── standards-kb/   # Transitive dependency
    └── templates-kb/   # Transitive dependency
```

This approach emerged from prior research and is documented in Decision 0005 (No Partial Resolution) and the Dependency Layout v2 specification. However, as the system matures, it's valuable to reconsider foundational decisions with fresh perspective.

## Current Implementation Strengths

### 1. Flat Layout Benefits

**Ergonomic Paths for Linking**
- Knowledge bases reference dependencies via `../.graft/meta-kb/docs/file.md`
- Short, predictable paths work from any location
- Stable regardless of dependency graph structure

**Deduplication**
```
Scenario: meta-kb and docs-kb both depend on templates-kb

Current system:
.graft/templates-kb/  ← Single clone, shared by both

Alternative (nested):
.graft/meta-kb/.graft/templates-kb/   ← First copy
.graft/docs-kb/.graft/templates-kb/   ← Duplicate copy
```

**Explicit Conflict Detection**
- Same dependency at different versions causes immediate failure
- No silent incompatibilities or version confusion
- Clear error messages guide resolution

### 2. Lock File as Source of Truth

The lock file documents the **complete dependency graph**:
```yaml
dependencies:
  meta-kb:
    source: "https://github.com/org/meta-kb.git"
    ref: "v2.0.0"
    commit: "abc123..."
    direct: true
    requires: ["standards-kb"]
    required_by: []

  standards-kb:
    source: "https://github.com/org/standards-kb.git"
    ref: "v1.5.0"
    commit: "def456..."
    direct: false
    requires: ["templates-kb"]
    required_by: ["meta-kb"]
```

Benefits:
- Complete provenance tracking
- Queryable dependency relationships
- Metadata-rich (timestamps, direct vs transitive)
- Supports tooling and AI analysis

### 3. Implementation Flexibility

- Custom checkout locations with symlinks for stable paths
- Can optimize with git sparse-checkout or shallow clones
- Not constrained by git submodule semantics
- Future-proof for extensions

## Alternative 1: Git Submodules

### Proposed Approach

Replace direct checkouts with git submodules:
```
project/
├── .gitmodules
├── .git/
└── .graft/
    └── meta-kb/    # git submodule (direct dependency)
        └── .graft/
            └── standards-kb/  # nested submodule (transitive)
```

### Advantages

**Native Git Integration**
```bash
# Single command gets repo + all dependencies
git clone --recurse-submodules https://github.com/org/kb.git

# Standard git operations
git submodule update --remote
git submodule foreach 'git status'
```

**Version Control Integration**
- Submodule refs part of parent repo history
- `git diff` shows dependency changes
- GitHub/GitLab render submodule links
- Code review tools understand submodules

**Established Tooling**
- IDE support (VS Code, IntelliJ)
- CI/CD systems have built-in handling
- Familiar to developers

### Critical Challenges

**1. Transitive Dependencies Create Nested Structure**

Git submodules only track direct children. For transitive dependencies:
```bash
# Standard submodule commands only go one level
git submodule update --init

# Need recursive flag for transitives
git submodule update --init --recursive
```

But this creates nested structure:
```
.graft/meta-kb/.graft/standards-kb/
```

This **breaks the stable path requirement** - references like `../.graft/standards-kb/` don't work.

**2. Path Instability Without Flattening**

If `meta-kb` contains markdown:
```markdown
See [Terminology](../.graft/standards-kb/terminology.md)
```

But `standards-kb` is actually at `meta-kb/.graft/standards-kb/`, the link is broken.

**3. Shared Dependency Duplication**

```
.graft/
├── meta-kb/
│   └── .graft/
│       └── templates-kb/   ← First clone
└── docs-kb/
    └── .graft/
        └── templates-kb/   ← Duplicate clone
```

Submodules don't deduplicate - each parent gets its own copy. This wastes disk space and creates potential version skew.

**4. No Automatic Conflict Detection**

If `meta-kb` requires `standards-kb@v1.5.0` and `docs-kb` requires `standards-kb@v1.0.0`:
- Both versions installed in different locations
- No automatic detection of the conflict
- Potential for runtime confusion
- Hard to detect programmatically

### Hybrid: Submodules + Flattening

**Proposed Solution**: Use submodules but add flattening step
```bash
# After git submodule update --recursive
graft flatten

# Creates symlinks:
.graft/
├── meta-kb/              # submodule (direct)
│   └── .graft/
│       └── standards-kb/ # nested submodule (actual location)
└── standards-kb/         # symlink → meta-kb/.graft/standards-kb/
```

**Windows Symlink Concerns**

This approach requires symlinks, which are problematic on Windows:
- Require admin privileges (pre-Windows 10 v1703)
- Developer Mode required for unprivileged symlinks
- Not all Windows users have Developer Mode enabled

**Workarounds**:
- Junction points (Windows-specific, directories only)
- Hard links (files only, same volume)
- Copy instead of symlink (wastes space)
- Require Developer Mode (limits adoption)

**How other tools handle this**:
- **pnpm**: Hard links from global store, falls back to copy
- **Yarn**: Plug'n'Play avoids `node_modules` entirely
- **Bazel**: Uses junction points on Windows, symlinks on Unix

### Assessment

Git submodules would require:
1. Managing submodule semantics
2. Custom flattening step
3. Windows compatibility layer
4. Maintaining redundant lock file for metadata

**Complexity outweighs benefits.** The lock file would still be needed for graph metadata, making submodules redundant.

## Alternative 2: Artifact-Based Composition

### Conceptual Model

Instead of cloning transitive dependencies, upstream grafts create self-contained artifacts:

```
# meta-kb creates artifact including its dependencies
meta-kb-v2.0.0-artifact/
├── docs/                    # meta-kb's content
├── .bundled/
│   ├── standards-kb/        # Inlined transitive dependency
│   └── templates-kb/        # Inlined transitive dependency
└── .bundle-manifest.yaml    # What was included, versions
```

Consumers only clone direct dependencies:
```
your-kb/
└── .graft/
    └── meta-kb/   # Contains bundled transitives in .bundled/
```

### How It Would Work

**Artifact Generation**
```bash
$ cd meta-kb
$ graft bundle --output meta-kb-v2.0.0.bundle

# Process:
# 1. Clone all transitive dependencies
# 2. Copy into .bundled/ directory
# 3. Rewrite references: ../.graft/standards-kb/ → .bundled/standards-kb/
# 4. Create manifest documenting bundled versions
# 5. Commit and tag as release artifact
```

**Artifact Consumption**
```yaml
# graft.yaml
dependencies:
  meta-kb:
    source: "https://github.com/org/meta-kb.git"
    ref: "v2.0.0"
    artifact: true  # Consume as artifact, not source
```

```bash
$ graft resolve
# Only clones meta-kb (which contains bundled transitives)
# Does NOT clone standards-kb or templates-kb separately
```

### Advantages

**1. Smaller Dependency Surface**
- Only direct dependencies cloned
- Clearer security boundary (trust only what you explicitly list)
- Simpler trust model for consumers

**2. Simpler Installation**
- Fewer git operations
- Faster `graft resolve`
- Less disk space per project

**3. Immutability Guarantees**
- Artifact bundled at specific versions
- Even if transitive deps change, artifact stays stable
- No phantom dependency updates

**4. Clearer Dependency Declaration**
- If you use it, you list it
- No implicit access to transitives

### Critical Disadvantages

**1. Lost Source Access and Traceability**

```
Question: "Where did this terminology definition come from?"

Current model:
- Check .graft/standards-kb/.git history
- See original commits, authors, context
- Trace evolution over time

Artifact model:
- It's in .bundled/standards-kb/ with no git history
- Must manually lookup source repository
- Lost provenance and context
```

**2. Breaks Cross-Dependency Collaboration**

```
Scenario: You want to contribute a fix to standards-kb

Current model:
- Already cloned in .graft/standards-kb/
- Edit, test, commit, push
- Natural workflow

Artifact model:
- Not cloned - bundled inside meta-kb
- Must manually clone standards-kb separately
- Can't test changes in context
- Friction in contribution workflow
```

**3. Duplication When Multiple Dependencies Share Transitives**

```
Scenario: meta-kb and docs-kb both depend on templates-kb

Artifact model:
.graft/
├── meta-kb/
│   └── .bundled/templates-kb/  ← First copy
└── docs-kb/
    └── .bundled/templates-kb/  ← Duplicate copy

Problem: Duplication! Same content bundled multiple times
```

This violates the deduplication principle that the flat layout achieves.

**4. Broken Knowledge Graph Integrity**

From Decision 0005, knowledge bases cross-reference each other's dependencies:
```markdown
<!-- In your KB -->
See [Architecture Pattern](../.graft/meta-kb/patterns.md)

<!-- In meta-kb/patterns.md -->
Uses template from [Templates](../.graft/templates-kb/header.md)
```

**Current model**: Both references work - complete graph available

**Artifact model**: Second reference rewritten to `.bundled/templates-kb/`:
- Can't navigate directly to templates-kb source
- Can't see templates-kb's full context
- Lost semantic connection to source of truth

**5. Limited AI Agent Analysis**

From Graft's AI collaboration use case:
```
AI Task: "Analyze all architecture patterns across dependencies"

Current model:
- AI traverses .graft/*/docs/**/*.md
- Full git history available for context
- Can analyze dependency evolution

Artifact model:
- Only direct deps available as sources
- Transitive content bundled (no git context)
- Limited analysis depth
```

### Fundamental Question: What IS a Knowledge Base?

This choice depends on the nature of knowledge bases:

**If Knowledge Bases Are Like Libraries (npm packages)**
- Consumed, not modified
- Published artifacts make sense
- Clear versioning, immutability important
- Consumers trust published versions
- **→ Artifact model appropriate**

**If Knowledge Bases Are Like Wikis (living documents)**
- Collaborative, evolving
- Source access critical
- Traceability and provenance matter
- Contributors need context
- **→ Source model appropriate**

### Graft's Stated Vision

From architecture documentation, Graft is designed for:
- "Ecosystem of composable components" (modular)
- "Human + AI collaboration" (needs full context)
- "Evolvability through visible discovery" (needs source access)
- "Semantic dependency management" (tracks evolution, not snapshots)

**This aligns with the WIKI model** - living, composable knowledge that evolves over time.

### Artifact Model Conflicts With Explicit Architectural Decisions

**Decision 0005: No Partial Resolution** explicitly states:
> "Graft WILL NOT support partial dependency resolution."
>
> "All dependencies declared in `graft.yaml` and their complete transitive dependency graphs MUST be resolved together."

**Rationale cited**:
1. **Atomicity Violation** - Partial resolution creates partial states
2. **Reproducibility Violation** - Different environments resolve differently
3. **Explicitness Violation** - Unclear what "partial" means

The artifact model would create exactly this partial resolution:
- Only direct dependencies present as source
- Transitive dependencies bundled (not as sources)
- Different environments see different source availability
- Lost reproducibility of source access

### Assessment

The artifact model fundamentally misaligns with Graft's design philosophy:
- **Violates atomicity**: Creates incomplete source graph
- **Breaks composability**: Transitive deduplication impossible
- **Reduces traceability**: Lost git history and provenance
- **Hinders collaboration**: Contributors can't access full context
- **Limits AI agents**: Incomplete source graph for analysis

**However**, there IS a valid use case for artifacts in the future:

**Publishing Final Documentation**
```bash
# Development mode: source dependencies
graft resolve --mode=source
# → Full source access for development

# Production mode: bundled artifact
graft build --output=docs-bundle.tar.gz
# → Self-contained artifact for deployment
```

This hybrid approach:
- **Development**: Full source access, complete graph, contribution-friendly
- **Production**: Single artifact for deployment, immutable, self-contained

Similar to how Sphinx/Docusaurus work:
- Development: Live reloading from source files
- Production: Static HTML bundle for deployment

## Comparative Analysis: Modern Package Managers

### npm/yarn/pnpm (Node.js)

**npm v2**: Nested `node_modules` (like nested submodules)
- **Problem**: Duplication, deep paths, phantom dependencies

**npm v3+/yarn v1**: Hoisted/flattened `node_modules`
- **Solution**: Flat structure with deduplication
- **Similar to current Graft `.graft/` approach**

**pnpm**: Content-addressable store + hard links
- Global store at `~/.pnpm-store/`
- Project has `node_modules/.pnpm/` with hard links
- Solves deduplication without symlink issues

**Yarn v2+ (Berry)**: Plug'n'Play
- No `node_modules` directory
- Resolution map in `.pnp.cjs`
- Runtime resolves from zip files or cache

### Cargo (Rust)

- Global cache at `~/.cargo/registry/`
- Each project has `Cargo.lock` with exact versions
- Dependencies compiled from cache (no local copy)
- Lock file is canonical source of truth

### Go Modules

- Global cache at `$GOPATH/pkg/mod/`
- `go.mod` + `go.sum` lock dependencies
- No local copies - referenced from cache
- Flat dependency resolution with Minimum Version Selection

### Key Patterns Across Ecosystems

1. **Flat layouts win** - Nested dependencies cause problems
2. **Global cache + local references** - Avoid duplication across projects
3. **Lock files are canonical** - Record complete resolved state
4. **Symlinks optional** - Modern tools have fallbacks (hard links, copies)

## Conclusions

### Current Architecture Is Correct

After deep analysis, **transitive source cloning in flat layout is the right design** for Graft because:

1. **Knowledge bases are living, interconnected documents** - not immutable packages
2. **Cross-referencing across dependency boundaries is fundamental** - requires stable paths
3. **Traceability and source access are critical** - for collaboration and understanding
4. **The specification's design is coherent** - explicitly addresses these concerns
5. **Artifact model would break core principles** - atomicity, composability, deduplication

### Alignment With Core Principles

| Principle | Current Approach | Git Submodules | Artifact Model |
|-----------|-----------------|----------------|----------------|
| **Git-Native** | ✓ Uses git refs | ✓ Uses submodules | ⚠️ Bundles lose git context |
| **Explicit Over Implicit** | ✓ Complete lock file | ⚠️ Need lock file anyway | ❌ Partial source availability |
| **Atomic Operations** | ✓ All-or-nothing | ⚠️ With flattening | ❌ Creates partial states |
| **Reproducibility** | ✓ Complete graph | ⚠️ With custom tooling | ❌ Different source availability |
| **Composability** | ✓ Flat, deduplicated | ❌ Nested duplication | ❌ Bundled duplication |
| **Minimal Primitives** | ✓ Simple model | ❌ Adds submodule complexity | ❌ Adds artifact concept |

### Validation Against Specification

The exploration confirms that existing architectural decisions are well-founded:

- **Decision 0005 (No Partial Resolution)** - Validated by artifact model analysis
- **Dependency Layout v2 (Flat Layout)** - Validated by submodule comparison
- **Lock File v2 (Extended Fields)** - Validated by source traceability needs

## Future Enhancements

While maintaining the current architecture, these optimizations are valuable:

### Phase 1: Global Cache (High Priority)
```
~/.graft/cache/<commit-hash>/
```

Benefits:
- Reduce disk usage across projects
- Faster installation (cache hits)
- Hard links (not symlinks) to `.graft/` for compatibility

Similar to pnpm and Cargo patterns.

### Phase 2: Git Optimizations (Medium Priority)
- Shallow clones for dependencies (save bandwidth)
- Sparse checkout for large dependencies (only needed paths)
- Parallel clone operations (faster resolution)

### Phase 3: Optional Artifact Generation (Future)
```bash
graft bundle --output docs-bundle.tar.gz
```

For deployment, distribution, and archival use cases. **Not for development workflow.**

## Recommendations

1. **Continue with current architecture** - It is sound and well-aligned with goals
2. **Prioritize global cache** - Provides optimization without complexity
3. **Document the rationale** - This work log captures the analysis
4. **Consider hybrid artifact model** - For future deployment use cases only

## Open Questions (For Future)

### Q1: Global Cache Location
Should cache be:
- `~/.graft/cache/` (user-level)
- `.graft-cache/` in project (project-level)
- Configurable via environment variable

**Recommendation**: User-level by default, configurable

### Q2: Cache Eviction Strategy
When should cached dependencies be removed:
- Manual cleanup only
- LRU (least recently used)
- TTL (time to live)
- Configurable max size

**Recommendation**: Start with manual, add LRU later if needed

### Q3: Hard Link vs Copy Fallback
When hard links fail (cross-volume, unsupported filesystem):
- Error and require user action
- Silent fallback to copy
- Warning + fallback to copy

**Recommendation**: Warning + fallback (like pnpm)

## Next Steps

1. ✅ Complete this work log
2. Commit and push to branch
3. Create pull request
4. Archive for future reference

## Changelog

- **2026-01-12 Initial exploration**: Comprehensive analysis of:
  - Current `.graft/` checkout system
  - Git submodules alternative (with flattening considerations)
  - Artifact-based composition model
  - Alignment with Graft's design principles
  - Comparative analysis with modern package managers
  - **Conclusion**: Current architecture is correct; validated existing decisions
  - **Recommendation**: Proceed with global cache optimization, maintain current design
