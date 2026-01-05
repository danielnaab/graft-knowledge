---
title: "graft.yaml Format Specification"
date: 2026-01-01
status: draft
---

# graft.yaml Format Specification

## Overview

The `graft.yaml` file is the configuration file for Graft dependencies. It defines:
- Dependency metadata
- Changes (identified by git refs)
- Commands (migrations, verification, utilities)
- Dependencies on other Graft modules

This file lives in the root of a dependency repository and is the **source of truth for automation**.

## File Location

```
repository-root/
  graft.yaml          ← This file
  CHANGELOG.md        ← Optional human-readable changelog
  README.md
  src/
  codemods/
```

## Schema

### Top-Level Structure

```yaml
# Optional metadata
metadata:
  name: string                    # Dependency name
  description: string             # Brief description
  version: string                 # Current version (optional)
  changelog: string               # Path to CHANGELOG.md (default: "CHANGELOG.md")

# Change definitions (see Change Model spec)
changes:
  <git-ref>:
    type: string                  # Optional: "breaking", "feature", "fix", etc.
    description: string           # Optional: brief summary
    migration: string             # Optional: command name
    verify: string                # Optional: command name
    [custom-key]: any             # Optional: extensible metadata

# Command definitions
commands:
  <command-name>:
    run: string                   # Required: command to execute
    description: string           # Optional: human-readable description
    working_dir: string           # Optional: working directory (default: consumer root)
    env: object                   # Optional: environment variables

# Dependencies (for Graft-aware dependencies)
dependencies:
  <dep-name>:
    source: string                # Required: git URL or path
    ref: string                   # Optional: specific ref (default: main)

# Mirror configuration (optional, for enterprise/offline support)
mirrors:
  - pattern: string               # Required: glob pattern for URL matching
    replace: string               # Required: replacement URL pattern
    fallback: boolean             # Optional: try original if mirror fails (default: false)
```

## Section: metadata

Optional metadata about this dependency.

### Fields

#### name (optional)
**Type**: `string`

**Description**: Human-readable name of the dependency.

**Example**:
```yaml
metadata:
  name: "meta-knowledge-base"
```

#### description (optional)
**Type**: `string`

**Description**: Brief description of what this dependency provides.

**Example**:
```yaml
metadata:
  description: "Shared knowledge base for meta-cognitive patterns"
```

#### version (optional)
**Type**: `string`

**Description**: Current version. Informational only; actual version is determined by git refs.

**Example**:
```yaml
metadata:
  version: "2.0.0"
```

#### changelog (optional)
**Type**: `string`

**Description**: Path to human-readable changelog file (relative to repository root).

**Default**: `"CHANGELOG.md"`

**Example**:
```yaml
metadata:
  changelog: "CHANGELOG.md"
  changelog: "docs/RELEASES.md"
```

## Section: changes

Defines changes identified by git refs. See [Change Model Specification](./change-model.md) for detailed field definitions.

### Structure

```yaml
changes:
  <git-ref>:           # Key is the git ref (commit, tag, branch)
    type: string       # Optional
    description: string  # Optional
    migration: string  # Optional: command name
    verify: string     # Optional: command name
    [custom]: any      # Optional: extensible
```

### Example

```yaml
changes:
  v2.0.0:
    type: breaking
    description: "Renamed getUserData → fetchUserData"
    migration: migrate-v2
    verify: verify-v2

  v1.5.0:
    type: feature
    description: "Added caching support"
    # No migration needed

  abc123:
    type: fix
    migration: fix-abc
```

### Ordering

Changes are applied in **declaration order**. First change in the file is applied first.

**Important**: When upgrading from v1.0.0 to v3.0.0, list intermediate versions in order:

```yaml
changes:
  v1.0.0:
    migration: migrate-v1
  v2.0.0:
    migration: migrate-v2
  v3.0.0:
    migration: migrate-v3
```

## Section: commands

Defines executable commands that can be invoked by consumers or referenced by changes.

### Structure

```yaml
commands:
  <command-name>:          # Key is the command name
    run: string            # Required: shell command to execute
    description: string    # Optional: human-readable description
    working_dir: string    # Optional: working directory
    env:                   # Optional: environment variables
      KEY: value
```

### Fields

#### run (required)
**Type**: `string`

**Description**: Shell command to execute. Runs in consumer's context.

**Interpolation**: May use variables:
- `${CONSUMER_ROOT}`: Consumer's repository root
- `${DEP_ROOT}`: This dependency's root (if installed)

**Examples**:
```yaml
run: "npx jscodeshift -t codemods/v2.js src/"
run: "python migrations/migrate.py"
run: "./scripts/migrate.sh"
run: |
  npm test
  ./verify.sh
```

#### description (optional)
**Type**: `string`

**Description**: Human-readable description of what this command does.

**Example**:
```yaml
description: "Rename getUserData to fetchUserData"
```

#### working_dir (optional)
**Type**: `string`

**Description**: Working directory for command execution. Relative to consumer root.

**Default**: Consumer's repository root

**Example**:
```yaml
working_dir: "src/"
```

#### env (optional)
**Type**: `object` (key-value pairs)

**Description**: Environment variables to set during command execution.

**Example**:
```yaml
env:
  NODE_ENV: "production"
  MIGRATION_DRY_RUN: "false"
```

### Command Examples

#### Simple Migration

```yaml
commands:
  migrate-v2:
    run: "npx jscodeshift -t codemods/v2.js src/"
    description: "Rename getUserData → fetchUserData"
```

#### Multi-Step Migration

```yaml
commands:
  migrate-v3:
    run: |
      echo "Running migration v3..."
      ./scripts/step1.sh
      npx jscodeshift -t codemods/step2.js src/
      python scripts/step3.py
    description: "Multi-step migration for v3"
```

#### Migration with Verification

```yaml
commands:
  migrate-v2:
    run: "npx jscodeshift -t codemods/v2.js src/"

  verify-v2:
    run: |
      npm test
      ! grep -r 'getUserData' src/
    description: "Verify v2 migration: tests pass and no old API usage"
```

#### Conditional Migration

```yaml
commands:
  migrate-optional:
    run: |
      if [ -f "src/legacy.js" ]; then
        ./migrate-legacy.sh
      fi
    description: "Migrate legacy code if it exists"
```

## Section: dependencies

Declares dependencies on other Graft-enabled modules (optional).

### Structure

```yaml
dependencies:
  <dep-name>:
    source: string      # Required: git URL or path
    ref: string         # Optional: specific ref (default: main/master)
```

### Fields

#### source (required)
**Type**: `string`

**Description**: Git URL or local path to dependency repository.

**Formats**:
- SSH: `git@github.com:user/repo.git`
- HTTPS: `https://github.com/user/repo.git`
- Local: `../local-repo`

**Example**:
```yaml
source: "git@github.com:org/meta-kb.git"
```

#### ref (optional)
**Type**: `string`

**Description**: Specific git ref to use. If not specified, uses default branch.

**Example**:
```yaml
ref: "v1.5.0"
ref: "stable"
```

### Example

```yaml
dependencies:
  meta-knowledge-base:
    source: "git@github.com:org/meta-kb.git"
    ref: "v1.5.0"

  shared-utils:
    source: "../shared-utils"
```

## Section: mirrors

Configures URL rewriting for dependency sources (optional). Useful for enterprise environments, mirrors, and offline development.

### Structure

```yaml
mirrors:
  - pattern: string      # Required: glob pattern for URL matching
    replace: string      # Required: replacement URL pattern
    fallback: boolean    # Optional: try original if mirror fails (default: false)
```

### Fields

#### pattern (required)
**Type**: `string`

**Description**: Glob pattern to match against dependency source URLs. First matching pattern wins.

**Syntax**: Standard glob with `*` wildcard

**Examples**:
```yaml
pattern: "https://github.com/*"          # Match all github HTTPS URLs
pattern: "git@github.com:*"              # Match all github SSH URLs
pattern: "https://github.com/myorg/*"    # Match specific org only
```

#### replace (required)
**Type**: `string`

**Description**: Replacement URL pattern. `*` in pattern is replaced with matched portion.

**Examples**:
```yaml
# Replace github.com with internal mirror
pattern: "https://github.com/*"
replace: "https://github-mirror.corp.internal/*"

# Rewrite to different git server
pattern: "git@github.com:*"
replace: "git@git.corp.internal:mirrors/*"
```

#### fallback (optional)
**Type**: `boolean`

**Default**: `false`

**Description**: If true, attempt original URL if mirrored URL fails. Useful for hybrid environments (office + remote work).

**Examples**:
```yaml
# Try mirror first, fall back to original
fallback: true

# Mirror only (air-gapped environment)
fallback: false
```

### Semantics

1. **Pattern matching**: Patterns evaluated top-to-bottom, first match wins
2. **URL rewriting**: Matched `*` portion substituted into replacement
3. **Transparency**: Original URLs stored in lock file, not rewritten URLs
4. **Scope**: Project-level mirrors (in graft.yaml) override global mirrors (~/.graft/config.yaml)

### Examples

**Example 1: Corporate mirror**
```yaml
mirrors:
  - pattern: "https://github.com/*"
    replace: "https://github.corp-mirror.internal/*"
    fallback: true  # Work from office or home
```

**Example 2: Air-gapped environment**
```yaml
mirrors:
  - pattern: "https://github.com/*"
    replace: "https://internal-git.corp/mirrors/github/*"
    fallback: false  # No external access allowed

  - pattern: "git@github.com:*"
    replace: "git@internal-git.corp:mirrors/github/*"
    fallback: false
```

**Example 3: Fork substitution**
```yaml
# Use our fork instead of upstream
mirrors:
  - pattern: "https://github.com/upstream/old-kb.git"
    replace: "https://github.com/myorg/old-kb-fork.git"
    fallback: false
```

**Example 4: Local development**
```yaml
# Use local filesystem mirrors for speed
mirrors:
  - pattern: "https://github.com/myorg/*"
    replace: "file:///srv/git-mirrors/myorg/*"
    fallback: true  # Fetch from github if local mirror stale
```

### Notes

- Mirrors are transparent - lock file stores original URL, not mirror
- Global mirrors can be configured in `~/.graft/config.yaml` (same format)
- Project mirrors override global mirrors for matching patterns
- If no pattern matches, original URL used

See: [Decision 0007: Mirror and Offline Support](../decisions/decision-0007-mirror-support.md)

## Complete Example

```yaml
# graft.yaml - Complete example

metadata:
  name: "example-library"
  description: "Example library showing Graft integration"
  changelog: "CHANGELOG.md"

changes:
  v2.0.0:
    type: breaking
    description: "Renamed getUserData → fetchUserData"
    migration: migrate-v2
    verify: verify-v2
    jira_ticket: "LIB-123"

  v1.5.0:
    type: feature
    description: "Added caching support"
    # No migration needed

  v1.0.0:
    type: feature
    description: "Initial release"

commands:
  migrate-v2:
    run: "npx jscodeshift -t codemods/rename-getUserData.js src/"
    description: "Rename getUserData → fetchUserData"
    env:
      JSCODESHIFT_PARSER: "tsx"

  verify-v2:
    run: |
      npm test
      ! grep -r 'getUserData' src/
    description: "Verify v2 migration completed"

  changelog:
    run: "cat CHANGELOG.md"
    description: "Display changelog"

dependencies:
  meta-knowledge-base:
    source: "git@github.com:org/meta-kb.git"
    ref: "v1.5.0"
```

## Validation

### Schema Validation

```python
def validate_graft_yaml(config: dict) -> list[str]:
    """Validate graft.yaml structure. Returns list of errors."""
    errors = []

    # Validate changes section
    if 'changes' in config:
        if not isinstance(config['changes'], dict):
            errors.append("'changes' must be an object")
        else:
            for ref, change_data in config['changes'].items():
                # Validate migration references
                if 'migration' in change_data:
                    cmd = change_data['migration']
                    if 'commands' not in config or cmd not in config['commands']:
                        errors.append(f"Change '{ref}': migration '{cmd}' not found in commands")

                # Validate verify references
                if 'verify' in change_data:
                    cmd = change_data['verify']
                    if 'commands' not in config or cmd not in config['commands']:
                        errors.append(f"Change '{ref}': verify '{cmd}' not found in commands")

    # Validate commands section
    if 'commands' in config:
        if not isinstance(config['commands'], dict):
            errors.append("'commands' must be an object")
        else:
            for cmd_name, cmd_data in config['commands'].items():
                if 'run' not in cmd_data:
                    errors.append(f"Command '{cmd_name}': missing required 'run' field")

    # Validate dependencies section
    if 'dependencies' in config:
        if not isinstance(config['dependencies'], dict):
            errors.append("'dependencies' must be an object")
        else:
            for dep_name, dep_data in config['dependencies'].items():
                if 'source' not in dep_data:
                    errors.append(f"Dependency '{dep_name}': missing required 'source' field")

    return errors
```

### Git Ref Validation

```python
def validate_refs_exist(config: dict, repo_path: str) -> list[str]:
    """Validate that all refs in changes exist in git."""
    errors = []
    refs = set(config.get('changes', {}).keys())

    # Get all refs from git
    result = subprocess.run(
        ['git', 'show-ref'],
        cwd=repo_path,
        capture_output=True,
        text=True
    )

    available_refs = set()
    for line in result.stdout.splitlines():
        ref_name = line.split()[1]
        available_refs.add(ref_name.split('/')[-1])  # Get short name

    # Also get commit hashes
    result = subprocess.run(
        ['git', 'log', '--format=%H %h'],
        cwd=repo_path,
        capture_output=True,
        text=True
    )
    for line in result.stdout.splitlines():
        full_hash, short_hash = line.split()
        available_refs.add(full_hash)
        available_refs.add(short_hash)

    # Check each ref
    for ref in refs:
        if ref not in available_refs:
            errors.append(f"Ref '{ref}' does not exist in git repository")

    return errors
```

## CLI Validation

```bash
# Validate graft.yaml
$ graft validate

Validating graft.yaml...
✓ Schema is valid
✓ All migration commands exist
✓ All verify commands exist
✓ All refs exist in git repository
✓ All dependency sources are accessible

# Validate specific aspects
$ graft validate --schema-only
$ graft validate --refs-only
```

## Versioning

The graft.yaml format itself may evolve. Version can be specified:

```yaml
graft_version: "1.0"  # Optional: graft.yaml format version

metadata:
  name: "example"
```

If not specified, latest version is assumed.

## Related

- [Specification: Change Model](./change-model.md)
- [Specification: Lock File Format](./lock-file-format.md)
- [Specification: Core Operations](./core-operations.md)
- [Decision 0003: Explicit Change Declarations](../decisions/decision-0003-explicit-change-declarations.md)

## References

- YAML Specification: https://yaml.org/spec/
- Git refs: https://git-scm.com/book/en/v2/Git-Internals-Git-References
