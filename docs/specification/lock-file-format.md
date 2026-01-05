---
title: "Lock File Format Specification"
date: 2026-01-05
status: draft
version: 2.0
---

# Lock File Format Specification

## Overview

The `graft.lock` file tracks the exact state of **all** consumed dependencies (both direct and transitive). It records:
- Which dependencies are being used (direct and transitive)
- What version (git ref) has been consumed
- Resolved commit hash for integrity
- Dependency relationships (requires/required_by)
- When each dependency was last updated

This file should be committed to version control to ensure reproducible dependency states across environments.

## File Location

```
project-root/
  graft.yaml          ← Consumer's configuration
  graft.lock          ← This file (generated and updated by Graft)
  src/
  README.md
```

## Purpose

The lock file serves several purposes:

1. **Complete state tracking**: Records ALL dependencies (direct + transitive) and their versions
2. **Reproducibility**: Enables identical dependency states across machines
3. **Integrity**: Stores commit hashes to detect tampering
4. **Dependency graph**: Tracks relationships via requires/required_by fields
5. **History**: Can be tracked in git to see dependency evolution
6. **Atomicity**: Updated only when upgrade fully succeeds
7. **Garbage collection**: Identifies which deps are no longer needed

## Schema

### Top-Level Structure

```yaml
apiVersion: graft/v0

# All resolved dependencies (direct + transitive)
dependencies:
  <dep-name>:
    source: string           # Git URL or path
    ref: string              # Consumed git ref (tag, branch, commit)
    commit: string           # Resolved commit hash (SHA-1)
    consumed_at: datetime    # ISO 8601 timestamp
    direct: boolean          # Is this a direct dependency?
    requires: [string]       # List of dependencies this dep needs
    required_by: [string]    # List of dependencies that need this dep
```

### API Version

**Field**: `apiVersion` (required)

**Type**: `string`

**Description**: Identifies the lock file format version.

**Current value**: `graft/v0`

**Example**:
```yaml
apiVersion: graft/v0
```

**Note**: Currently in initial development phase. Format may evolve. Future versions will use `graft/v1`, `graft/v2`, etc. when the specification stabilizes.

## Section: dependencies

Maps dependency names to their current state.

### Ordering Convention

**Specification**: Dependencies SHOULD be ordered as follows:

1. **Direct dependencies first** - All dependencies with `direct: true`
2. **Transitive dependencies second** - All dependencies with `direct: false`
3. **Alphabetically within groups** - Within each group, sort by dependency name

**Rationale**: This conventional ordering improves human readability and makes git diffs more meaningful when dependencies change.

**Example**:
```yaml
dependencies:
  # Direct dependencies (alphabetical)
  docs-kb: {...}
  meta-kb: {...}

  # Transitive dependencies (alphabetical)
  standards-kb: {...}
  templates-kb: {...}
  utils-kb: {...}
```

**Note**: While this ordering is RECOMMENDED for generated lock files, parsers MUST accept dependencies in any order. This allows hand-editing when necessary without breaking validation.

See: [Decision 0005: Lock File Ordering Conventions](../decisions/decision-0005-lock-file-ordering.md)

### Fields

#### source (required)
**Type**: `string`

**Description**: Git URL or path to dependency repository. Must match the source in graft.yaml.

**Formats**:
- SSH: `git@github.com:user/repo.git`
- HTTPS: `https://github.com/user/repo.git`
- Local path: `../local-repo`

**Example**:
```yaml
source: "git@github.com:org/meta-kb.git"
```

#### ref (required)
**Type**: `string`

**Description**: The git ref that has been consumed. This is the version the consumer has integrated.

**Values**: Any valid git ref (commit hash, tag, branch)

**Semantics**: The consumer has applied all changes up to and including this ref.

**Example**:
```yaml
ref: "v1.5.0"         # Semver tag
ref: "abc123def456"   # Commit hash
ref: "release-2026-01"  # Date-based tag
```

#### commit (required)
**Type**: `string`

**Description**: The full commit hash that `ref` resolves to. Used for integrity verification.

**Format**: 40-character SHA-1 hash

**Purpose**:
- Detect if ref has been moved (e.g., branch advanced)
- Verify dependency hasn't been tampered with
- Enable exact reproduction

**Example**:
```yaml
commit: "abc123def456789012345678901234567890abcd"
```

#### consumed_at (required)
**Type**: `string` (ISO 8601 datetime)

**Description**: Timestamp when this version was last consumed/upgraded.

**Format**: `YYYY-MM-DDTHH:MM:SS[.mmmmmm][+HH:MM]`

**Example**:
```yaml
consumed_at: "2026-01-01T10:30:00Z"
consumed_at: "2026-01-01T10:30:00.123456+00:00"
```

#### direct (required)
**Type**: `boolean`

**Description**: Whether this is a direct dependency (declared in graft.yaml) or transitive (required by another dependency).

**Values**:
- `true`: Direct dependency (declared in consumer's graft.yaml)
- `false`: Transitive dependency (pulled in by another dep)

**Purpose**:
- Distinguish between deps you declared vs deps that are pulled in
- Enable filtering (e.g., "show only my direct deps")
- Guide upgrade decisions (direct deps vs transitive)

**Example**:
```yaml
direct: true   # You declared this
direct: false  # Pulled in transitively
```

#### requires (required)
**Type**: `list[string]`

**Description**: List of dependency names that this dependency requires (its own dependencies from its graft.yaml).

**Values**: List of dependency names, empty list if leaf dependency

**Purpose**:
- Reconstruct dependency tree
- Understand dependency chains
- Detect circular dependencies
- Plan upgrade impact

**Example**:
```yaml
requires: ["standards-kb", "templates-kb"]  # This dep needs these
requires: []                                  # Leaf dependency
```

#### required_by (required)
**Type**: `list[string]`

**Description**: List of dependency names that require this dependency.

**Values**: List of dependency names, empty list for direct dependencies

**Purpose**:
- Identify shared dependencies (multiple entries)
- Garbage collection (no entries means unused)
- Impact analysis ("what depends on this?")
- Upgrade planning

**Example**:
```yaml
required_by: []                   # Direct dep (nothing requires it from us)
required_by: ["meta-kb"]          # Only meta-kb needs this
required_by: ["meta-kb", "docs-kb"]  # Shared by both!
```

## Complete Example

### Simple Project (one direct dep with transitive deps)

```yaml
apiVersion: graft/v0

dependencies:
  # Direct dependency
  meta-knowledge-base:
    source: "git@github.com:org/meta-kb.git"
    ref: "v2.0.0"
    commit: "abc123def456789012345678901234567890abcd"
    consumed_at: "2026-01-05T10:30:00Z"
    direct: true
    requires: ["standards-kb"]
    required_by: []

  # Transitive dependency (from meta-kb)
  standards-kb:
    source: "https://github.com/org/standards.git"
    ref: "v1.5.0"
    commit: "def456abc123789012345678901234567890abcd"
    consumed_at: "2026-01-05T10:30:00Z"
    direct: false
    requires: ["templates-kb"]
    required_by: ["meta-knowledge-base"]

  # Transitive dependency (from standards-kb)
  templates-kb:
    source: "https://github.com/org/templates.git"
    ref: "v1.0.0"
    commit: "789abc456def012345678901234567890abcdef12"
    consumed_at: "2026-01-05T10:30:00Z"
    direct: false
    requires: []
    required_by: ["standards-kb"]
```

### Complex Project (shared dependencies)

```yaml
apiVersion: graft/v0

dependencies:
  # Direct dependency #1
  meta-kb:
    source: "git@github.com:org/meta-kb.git"
    ref: "v2.0.0"
    commit: "abc123..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: true
    requires: ["templates-kb"]
    required_by: []

  # Direct dependency #2
  docs-kb:
    source: "git@github.com:org/docs-kb.git"
    ref: "v1.0.0"
    commit: "bcd234..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: true
    requires: ["templates-kb"]
    required_by: []

  # Shared transitive dependency
  templates-kb:
    source: "https://github.com/org/templates.git"
    ref: "v1.0.0"
    commit: "def456..."
    consumed_at: "2026-01-05T10:30:00Z"
    direct: false
    requires: []
    required_by: ["meta-kb", "docs-kb"]  # Shared!
```

**Note:** `templates-kb` appears in multiple `required_by` lists, indicating it's shared.

## Lifecycle

### Creation

Generated when first dependency is added:

```bash
$ graft add meta-kb --source git@github.com:org/meta-kb.git --ref v1.0.0

Created graft.lock:
  meta-kb@v1.0.0
```

### Updates

Updated when dependency is upgraded:

```bash
$ graft upgrade meta-kb --to v2.0.0

Updated graft.lock:
  meta-kb: v1.0.0 → v2.0.0
```

**Important**: Lock file is only updated when upgrade fully succeeds (atomic operation).

### Manual Editing

Generally not recommended. Use `graft` commands instead.

If manual editing is necessary:
- Ensure YAML is valid
- Update all fields together (ref + commit + consumed_at)
- Run `graft validate` to check consistency

## Comparison to Other Lock Files

### Similar to package-lock.json (npm)

```json
{
  "dependencies": {
    "package-name": {
      "version": "1.0.0",
      "resolved": "https://...",
      "integrity": "sha512-..."
    }
  }
}
```

**Similarities**:
- Tracks exact versions
- Committed to version control
- Enables reproducibility

**Differences**:
- Graft uses git refs, not npm versions
- Graft tracks consumption state, not just installation
- Graft uses commit hash for integrity, not content hash

### Similar to Cargo.lock (Rust)

```toml
[[package]]
name = "package-name"
version = "1.0.0"
source = "registry+https://..."
checksum = "abc123..."
```

**Similarities**:
- Declarative format
- Checksum for integrity
- Version pinning

**Differences**:
- Graft uses git refs directly
- Graft designed for knowledge/code dependencies, not just code libraries

## Validation

### Lock File Validation

```python
def validate_lock_file(lock: dict) -> list[str]:
    """Validate lock file structure and content."""
    errors = []

    # Check version
    if 'version' not in lock:
        errors.append("Missing 'version' field")
    elif lock['version'] != 1:
        errors.append(f"Unsupported lock file version: {lock['version']}")

    # Check dependencies
    if 'dependencies' not in lock:
        errors.append("Missing 'dependencies' section")
        return errors

    for dep_name, dep_data in lock['dependencies'].items():
        # Required fields
        for field in ['source', 'ref', 'commit', 'consumed_at']:
            if field not in dep_data:
                errors.append(f"Dependency '{dep_name}': missing '{field}'")

        # Validate commit hash format
        if 'commit' in dep_data:
            commit = dep_data['commit']
            if not re.match(r'^[0-9a-f]{40}$', commit):
                errors.append(f"Dependency '{dep_name}': invalid commit hash '{commit}'")

        # Validate timestamp format
        if 'consumed_at' in dep_data:
            try:
                datetime.fromisoformat(dep_data['consumed_at'].replace('Z', '+00:00'))
            except ValueError:
                errors.append(f"Dependency '{dep_name}': invalid timestamp format")

    return errors
```

### Integrity Verification

```python
def verify_integrity(lock: dict, dep_name: str, repo_path: str) -> bool:
    """Verify that ref resolves to expected commit."""
    dep_data = lock['dependencies'][dep_name]
    expected_commit = dep_data['commit']

    # Resolve ref to commit
    result = subprocess.run(
        ['git', 'rev-parse', dep_data['ref']],
        cwd=repo_path,
        capture_output=True,
        text=True
    )

    actual_commit = result.stdout.strip()

    if actual_commit != expected_commit:
        print(f"Warning: {dep_name} ref '{dep_data['ref']}' has moved!")
        print(f"  Expected: {expected_commit}")
        print(f"  Actual:   {actual_commit}")
        return False

    return True
```

## Operations

### Read Lock File

```python
def read_lock_file(path: str = 'graft.lock') -> dict:
    """Read and parse lock file."""
    with open(path) as f:
        return yaml.safe_load(f)
```

### Update Dependency

```python
def update_lock_file(
    dep_name: str,
    ref: str,
    commit: str,
    source: str,
    lock_path: str = 'graft.lock'
):
    """Update lock file after successful upgrade."""
    lock = read_lock_file(lock_path)

    if 'dependencies' not in lock:
        lock['dependencies'] = {}

    lock['dependencies'][dep_name] = {
        'source': source,
        'ref': ref,
        'commit': commit,
        'consumed_at': datetime.now(timezone.utc).isoformat()
    }

    with open(lock_path, 'w') as f:
        yaml.dump(lock, f, default_flow_style=False, sort_keys=False)
```

### Get Current Version

```python
def get_consumed_ref(dep_name: str, lock_path: str = 'graft.lock') -> Optional[str]:
    """Get currently consumed ref for a dependency."""
    lock = read_lock_file(lock_path)
    dep_data = lock.get('dependencies', {}).get(dep_name)
    return dep_data['ref'] if dep_data else None
```

## CLI Integration

```bash
# Show lock file status
$ graft status
Dependencies:
  meta-kb: v1.5.0 (consumed 2026-01-01)
  shared-utils: v2.0.0 (consumed 2025-12-15)

# Check for updates
$ graft status --check-updates
Dependencies:
  meta-kb: v1.5.0 → v2.0.0 available
  shared-utils: v2.0.0 (up to date)

# Validate lock file
$ graft validate --lock
Lock file is valid
All commits verified
No integrity issues
```

## Git Integration

The lock file should be committed to git:

```bash
# After upgrade
$ graft upgrade meta-kb --to v2.0.0
Upgraded meta-kb to v2.0.0

$ git status
modified:   graft.lock

$ git add graft.lock
$ git commit -m "Upgrade meta-kb to v2.0.0"
```

This creates a history of dependency evolution:

```bash
$ git log --oneline -- graft.lock
abc123 Upgrade meta-kb to v2.0.0
def456 Upgrade shared-utils to v2.0.0
789abc Initial graft.lock
```

## Edge Cases

### Ref Has Moved (Branch Updated)

```yaml
# Lock file says:
dependencies:
  meta-kb:
    ref: "main"
    commit: "abc123"

# But main has advanced to def456
```

**Detection**:
```bash
$ graft validate --lock
Warning: meta-kb ref 'main' has moved
  Lock file: abc123
  Current:   def456
  Run 'graft upgrade meta-kb' to update
```

**Resolution**: Run upgrade to update to new commit.

### Deleted Ref

```yaml
# Lock file references:
dependencies:
  meta-kb:
    ref: "feature-branch"
    commit: "abc123"

# But feature-branch was deleted
```

**Detection**:
```bash
$ graft validate --lock
Error: meta-kb ref 'feature-branch' does not exist
  Commit abc123 is still accessible
  Consider updating to a stable ref (tag or main)
```

**Resolution**: Update to a different ref that points to the same commit or newer.

### Source URL Changed

```yaml
# Lock file:
source: "git@github.com:old-org/repo.git"

# graft.yaml now says:
source: "git@github.com:new-org/repo.git"
```

**Detection**:
```bash
$ graft validate
Warning: meta-kb source URL differs between graft.yaml and graft.lock
  Lock: git@github.com:old-org/repo.git
  Config: git@github.com:new-org/repo.git
```

**Resolution**: Update lock file source to match graft.yaml.

## Lock File History (Optional Future Enhancement)

Could optionally include history:

```yaml
version: 1

dependencies:
  meta-kb:
    source: "git@github.com:org/meta-kb.git"
    ref: "v2.0.0"
    commit: "def456..."
    consumed_at: "2026-01-01T10:30:00Z"
    history:
      - ref: "v1.0.0"
        commit: "abc123..."
        consumed_at: "2025-10-01T08:00:00Z"
      - ref: "v1.5.0"
        commit: "bcd234..."
        consumed_at: "2025-12-15T09:00:00Z"
```

**Not currently specified** - use git log on graft.lock instead.

## Related

- [Specification: graft.yaml Format](./graft-yaml-format.md)
- [Specification: Core Operations](./core-operations.md)
- [Decision 0004: Atomic Upgrades](../decisions/decision-0004-atomic-upgrades.md)

## References

- YAML Specification: https://yaml.org/spec/
- ISO 8601 (datetime format): https://en.wikipedia.org/wiki/ISO_8601
- Git commit hashing: https://git-scm.com/book/en/v2/Git-Internals-Git-Objects
