---
title: "Decision 0008: API Versioning Semantics"
date: 2026-01-05
status: accepted
---

# Decision 0008: API Versioning Semantics

## Context

Graft specifications (graft.yaml and graft.lock) include an `apiVersion` field to version the file formats. However, the semantics of version numbers and how they evolve have not been clearly specified.

Questions that need answers:
- What does `graft/v0` vs `graft/v1` mean?
- When should version numbers change?
- How do tools handle multiple versions?
- What backward compatibility guarantees exist?

Without clear versioning semantics, the ecosystem risks:
- Incompatible implementations
- Confusion about when breaking changes occur
- Difficult migrations between versions
- Tool fragmentation

## Decision

**Adopt Kubernetes-style API versioning with clear stability guarantees.**

### Version Format

`apiVersion: graft/v{N}`

Where:
- `graft/` is the namespace (allows future alternate namespaces if needed)
- `v{N}` is the major version number (v0, v1, v2, ...)

### Version Semantics

#### `graft/v0` - Experimental/Preview

**Stability**: No backward compatibility guarantees

**Semantics**:
- Breaking changes MAY occur between v0 releases
- Used during initial development and specification
- Signals: "Expect changes, provide feedback"
- Tools MAY warn users about experimental status

**Allowed changes without version bump**:
- Adding optional fields
- Changing field semantics
- Removing fields
- Changing validation rules

**Duration**: Until specification and reference implementation stabilize

#### `graft/v1` - First Stable Release

**Stability**: Strong backward compatibility within v1.x

**Semantics**:
- Breaking changes MUST NOT occur within v1
- Backward-compatible additions allowed (new optional fields)
- Tools supporting v1 MUST handle all v1.x formats
- Removal of v0 support allowed after migration period

**Allowed changes (backward compatible)**:
- Adding new optional fields
- Adding new optional sections
- Relaxing validation (accepting more)
- Clarifying semantics (without changing behavior)

**Forbidden changes (breaking)**:
- Removing required fields
- Removing optional fields (must deprecate first)
- Changing field types
- Changing required field semantics
- Strictening validation (rejecting previously valid files)

**Deprecation process**:
1. Mark field as deprecated in specification (docs)
2. Tools MAY warn when deprecated field used
3. Field remains valid for entire v1 lifecycle
4. Field can be removed in v2

#### `graft/v2+` - Subsequent Major Versions

**Stability**: Each major version has same guarantees as v1

**Semantics**:
- Signals breaking changes from previous version
- Migration guide SHOULD be provided
- Tools MAY support multiple versions simultaneously
- Previous version supported for reasonable transition period

**Examples of breaking changes warranting v2**:
- Removing previously deprecated fields
- Fundamental restructuring (e.g., flat to nested)
- Changing core semantics (e.g., dependency resolution algorithm)
- Incompatible schema changes

### Tool Compatibility Requirements

**1. Version Detection**

Tools MUST:
- Read `apiVersion` field first
- Reject unsupported versions with clear error
- Suggest upgrade path if available

```python
def parse_lock_file(path: str) -> LockFile:
    data = yaml.safe_load(open(path))
    version = data.get("apiVersion")

    if version not in SUPPORTED_VERSIONS:
        raise UnsupportedVersionError(
            f"Lock file version {version} not supported.\n"
            f"Supported versions: {', '.join(SUPPORTED_VERSIONS)}\n"
            f"Upgrade tool to support {version} or migrate file."
        )

    return parse_version_specific(data, version)
```

**2. Multi-Version Support (Optional)**

Tools MAY support multiple versions:

```python
SUPPORTED_VERSIONS = ["graft/v0", "graft/v1"]

def parse_lock_file(path: str) -> LockFile:
    version = detect_version(path)
    parser = get_parser(version)  # Version-specific parser
    return parser.parse(path)
```

**3. Migration Tooling**

When releasing new major version, tools SHOULD provide migration:

```bash
# Migrate v0 → v1
graft migrate-lock --from v0 --to v1

# Preview changes without writing
graft migrate-lock --dry-run
```

## Rationale

### Why Kubernetes-style versioning?

**Proven approach**:
- Used successfully in Kubernetes for 10+ years
- Clear stability signals (alpha, beta, v1, v2)
- Well-understood in devops community

**Simple**: Just major versions, no minor/patch complexity

**Clear guarantees**: Unambiguous what each version promises

### Why no minor/patch versions?

Configuration files are different from software versions:

**Files are data, not code**:
- Backward-compatible additions don't need new version
- Adding optional field in v1.2 vs v1.0 - both are v1
- Parsers ignore unknown fields → natural forward compatibility

**Semantic Versioning adds complexity**:
- v1.2.3 vs v1.2.4 - does parser care? No.
- Patch versions for file formats are meaningless
- Minor versions blur stability guarantees

**Industry precedent**:
- Kubernetes: Just v1, v2, v1alpha1, v1beta1
- Docker Compose: Just v1, v2, v3
- OpenAPI: Just 2.0, 3.0, 3.1

### Why `graft/v0` for experimental?

**Signals instability explicitly**:
- v0 is universally understood as "not stable yet"
- Tools can warn users appropriately
- Lowers expectations, encourages feedback

**Prevents premature v1**:
- Rushing to v1 locks in mistakes
- v0 gives freedom to iterate on design
- Can make breaking changes freely

**Aligns with semantic versioning culture**:
- Many tools use 0.x for pre-1.0 development
- Developers understand v0 implications

## Alternatives Considered

### Alternative 1: Semantic Versioning (v1.2.3)

Use full semver for file format versions:

```yaml
apiVersion: graft/v1.2.3
```

**Rejected because**:
- Unnecessary complexity for configuration files
- Minor/patch versions don't add value
- Parsers don't care about patch differences
- Blurs stability boundaries (is v1.9.0 → v2.0.0 a big change?)

### Alternative 2: Date-Based Versioning

```yaml
apiVersion: graft/2026-01
```

**Rejected because**:
- Unclear what changes between dates
- No stability signal
- Harder to determine compatibility
- When does 2026-01 become incompatible with 2026-02?

### Alternative 3: No Versioning

Assume forward/backward compatibility always.

**Rejected because**:
- Impossible to make breaking changes
- Locks in early design mistakes
- No way to signal major improvements
- Ecosystem stagnates or fractures

### Alternative 4: Separate File Type Versions

Different versions for graft.yaml vs graft.lock:

```yaml
# graft.yaml
configVersion: graft/v1

# graft.lock
lockVersion: graft/v2
```

**Rejected because**:
- Complexity - two version namespaces
- Confusion - which version am I using?
- Unnecessary - versions can evolve independently already

## Consequences

### Positive

- **Clear stability signals**: Users know what to expect from each version
- **Safe evolution**: Can make breaking changes via major versions
- **Simple**: Just major versions, easy to understand
- **Proven**: Based on successful Kubernetes model

### Negative

- **Breaking changes disruptive**: v0 → v1 transition requires migration
  - *Mitigated by*: Provide migration tooling
- **Multiple version support**: Tools may need to handle v0 and v1
  - *Mitigated by*: Can drop v0 after transition period

### Neutral

- **Documentation burden**: Must clearly explain version semantics
- **Specification overhead**: Need to track what version changes occurred

## Implementation

### Phase 1: Document v0 Status (Current)

- Update all specifications with version semantics
- Add warnings about v0 experimental status
- Document expected v1 timeline

### Phase 2: Stabilize for v1

- Gather feedback on v0 implementation
- Identify any needed breaking changes
- Finalize v1 specification
- Write v0 → v1 migration guide

### Phase 3: Release v1

- Publish v1 specifications
- Release reference tool with v1 support
- Provide `graft migrate-lock` command
- Support both v0 and v1 during transition (6-12 months)

### Phase 4: Deprecate v0

- Announce v0 deprecation timeline
- Warn users on v0 lock files
- Eventually remove v0 support (after transition period)

## Version Compatibility Matrix

### Current State

| Tool Version | graft/v0 Support | graft/v1 Support |
|-------------|-----------------|-----------------|
| 0.1.x       | ✅ Full         | ❌ Not yet      |
| 0.2.x       | ✅ Full         | ❌ Not yet      |

### Future State (Post v1 Release)

| Tool Version | graft/v0 Support | graft/v1 Support |
|-------------|-----------------|-----------------|
| 1.0.x       | ✅ Full         | ✅ Full         |
| 1.1.x       | ⚠️ Deprecated  | ✅ Full         |
| 2.0.x       | ❌ Removed     | ✅ Full         |

## Migration Example

### v0 → v1 Changes (Hypothetical)

```yaml
# v0 format
apiVersion: graft/v0
dependencies:
  meta-kb:
    source: "..."
    ref: "v2.0.0"
    commit: "abc123"
    consumed_at: "2026-01-01T10:00:00Z"
    direct: true
    requires: ["standards-kb"]
    required_by: []

# v1 format (if we changed structure)
apiVersion: graft/v1
dependencies:
  - name: meta-kb  # Changed from map to list
    source: "..."
    version:       # Nested version info
      ref: "v2.0.0"
      commit: "abc123"
      consumed_at: "2026-01-01T10:00:00Z"
    metadata:
      direct: true
      depends_on: ["standards-kb"]  # Renamed from requires
```

Migration tool handles conversion automatically:

```bash
$ graft migrate-lock --from v0 --to v1

Analyzing graft.lock (v0)...
Converting to v1 format...
  - Restructured dependencies section
  - Renamed 'requires' → 'depends_on'
  - Nested version fields
Writing graft.lock (v1)...
✓ Migration complete

Backup saved: graft.lock.v0.backup
```

## Related

- [Lock File Format Specification](../specification/lock-file-format.md) - Uses apiVersion
- [graft.yaml Format Specification](../specification/graft-yaml-format.md) - Also versioned
- [Decision 0003: Explicit Change Declarations](./decision-0003-explicit-change-declarations.md) - Explicit over implicit

## References

- **Kubernetes API Versioning**: https://kubernetes.io/docs/reference/using-api/#api-versioning
- **Semantic Versioning**: https://semver.org/ (for comparison)
- **Docker Compose versions**: https://docs.docker.com/compose/compose-file/compose-versioning/
- **OpenAPI versioning**: https://spec.openapis.org/oas/latest.html#versions
