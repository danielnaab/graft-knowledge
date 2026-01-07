---
title: "Core Operations Specification"
date: 2026-01-01
status: draft
---

# Core Operations Specification

## Overview

This document specifies the core operations that Graft provides for managing dependency changes and upgrades.

Operations are divided into:
- **Query operations**: Read-only, inspect state
- **Mutation operations**: Modify state (lock file, files)

## Query Operations

### graft status

**Purpose**: Show current state of dependencies and available updates.

**Syntax**:
```bash
graft status [<dep-name>] [options]
```

**Options**:
- `--json`: Output as JSON
- `--check-updates`: Fetch latest from upstream and show available updates

**Behavior**:
1. Read graft.lock to get current consumed versions
2. Optionally fetch latest from upstream (if --check-updates)
3. Display current version and available updates for each dependency

**Output** (text):
```
Dependencies:
  meta-kb: v1.5.0 (consumed 2026-01-01)
  shared-utils: v2.0.0 → v2.1.0 available (1 feature)
```

**Output** (JSON):
```json
{
  "dependencies": {
    "meta-kb": {
      "current": "v1.5.0",
      "consumed_at": "2026-01-01T10:30:00Z",
      "available": null
    },
    "shared-utils": {
      "current": "v2.0.0",
      "consumed_at": "2025-12-15T14:20:00Z",
      "available": "v2.1.0",
      "pending_changes": 1
    }
  }
}
```


---

### graft changes

**Purpose**: List changes for a dependency.

**Syntax**:
```bash
graft changes <dep-name> [options]
```

**Options**:
- `--from <ref>`: Start ref (default: current consumed version)
- `--to <ref>`: End ref (default: latest)
- `--since <ref>`: Alias for `--from <ref> --to latest`
- `--type <type>`: Filter by type (breaking, feature, fix, etc.)
- `--breaking`: Show only breaking changes
- `--format <format>`: Output format (text, json)

**Behavior**:
1. Read dependency's graft.yaml
2. Determine ref range (from current consumed version to latest, or as specified)
3. Filter changes in that range
4. Optionally filter by type
5. Display changes

**Output** (text):
```
Changes for meta-kb: v1.5.0 → v2.0.0

v2.0.0 (breaking)
  Renamed getUserData → fetchUserData
  Migration: migrate-v2
  Verify: verify-v2

v1.6.0 (feature)
  Added response caching
  No migration required
```

**Output** (JSON):
```json
{
  "dependency": "meta-kb",
  "from": "v1.5.0",
  "to": "v2.0.0",
  "changes": [
    {
      "ref": "v1.6.0",
      "type": "feature",
      "description": "Added response caching",
      "migration": null,
      "verify": null
    },
    {
      "ref": "v2.0.0",
      "type": "breaking",
      "description": "Renamed getUserData → fetchUserData",
      "migration": "migrate-v2",
      "verify": "verify-v2"
    }
  ]
}
```


---

### graft show

**Purpose**: Show details of a specific change.

**Syntax**:
```bash
graft show <dep-name>@<ref> [options]
```

**Options**:
- `--format <format>`: Output format (text, json)
- `--field <field>`: Show only specific field (migration, verify, etc.)

**Behavior**:
1. Load dependency's graft.yaml
2. Find change for specified ref
3. Display full details

**Output** (text):
```
Change: meta-kb@v2.0.0

Type: breaking
Description: Renamed getUserData → fetchUserData

Migration: migrate-v2
  Command: npx jscodeshift -t codemods/v2.js src/
  Description: Rename getUserData to fetchUserData

Verification: verify-v2
  Command: npm test && ! grep -r 'getUserData' src/
  Description: Verify v2 migration: tests pass and no old API usage

See CHANGELOG.md for full details and rationale.
```

**Output** (JSON):
```json
{
  "ref": "v2.0.0",
  "type": "breaking",
  "description": "Renamed getUserData → fetchUserData",
  "migration": "migrate-v2",
  "verify": "verify-v2",
  "migration_command": {
    "name": "migrate-v2",
    "run": "npx jscodeshift -t codemods/v2.js src/",
    "description": "Rename getUserData to fetchUserData"
  },
  "verify_command": {
    "name": "verify-v2",
    "run": "npm test && ! grep -r 'getUserData' src/",
    "description": "Verify v2 migration: tests pass and no old API usage"
  }
}
```


---

### graft fetch

**Purpose**: Update local cache of dependency's remote state.

**Syntax**:
```bash
graft fetch [<dep-name>]
```

**Behavior**:
1. For each dependency (or specified dependency)
2. Fetch latest from remote repository
3. Update local cache of available refs and changes
4. Do not modify lock file or consumed version

**Output**:
```
Fetching meta-kb...
  ✓ Fetched latest from git@github.com:org/meta-kb.git
  Latest: v2.0.0

Fetching shared-utils...
  ✓ Fetched latest from ../shared-utils
  Latest: v2.1.0
```


---

## Resolution Operations

### graft resolve

**Purpose**: Clone or fetch dependencies specified in graft.yaml.

**Syntax**:
```bash
graft resolve
```

**Behavior**:
1. Find and parse graft.yaml in current directory
2. For each dependency:
   - If `.graft/<name>/` doesn't exist: clone from git URL
   - If `.graft/<name>/` exists and is a git repo: fetch and checkout ref
   - If `.graft/<name>/` exists but isn't a git repo: error
3. Report resolution status for each dependency
4. Auto-add `.graft` to `.gitignore` if not already present

**Output** (success):
```
Found configuration: /path/to/project/graft.yaml
API Version: graft/v0
Dependencies: 2

Resolving dependencies...

✓ meta-kb: resolved to /path/to/project/.graft/meta-kb
✓ standards-kb: resolved to /path/to/project/.graft/standards-kb

Resolved: 2/2

All dependencies resolved successfully!
```

**Output** (failure):
```
Found configuration: /path/to/project/graft.yaml
API Version: graft/v0
Dependencies: 2

Resolving dependencies...

✓ meta-kb: resolved to /path/to/project/.graft/meta-kb
✗ standards-kb: Authentication failed
  Suggestion: Check SSH key configuration

Resolved: 1/2

Some dependencies failed to resolve.
```

**Exit codes**:
- `0` - All dependencies resolved
- `1` - One or more dependencies failed

**Notes**:
- Dependencies are stored in `.graft/<name>/` (flat layout)
- Paths in output are absolute for clarity
- `.graft` is auto-added to `.gitignore` to prevent committing dependencies
- When using a custom `deps_directory`, symlinks are created in `.graft/<name>` pointing to the actual checkout location, ensuring stable paths for linking


---

## Validation Operations

### graft validate

**Purpose**: Validate graft configuration files and dependency integrity.

**Syntax**:
```bash
graft validate [mode] [options]
```

**Modes**:
- `config` - Validate graft.yaml syntax and semantics
- `lock` - Validate graft.lock format and consistency
- `integrity` - Verify .graft/ directory matches lock file
- `all` - Run all validations (default)

**Options**:
- `--json`: Output as JSON
- `--fix`: Attempt to fix issues automatically (where possible)

**Behavior**:

**Mode: config**
1. Parse graft.yaml as YAML
2. Check required fields present
3. Validate git URLs format
4. Check command references are valid

**Mode: lock**
1. Parse graft.lock as YAML
2. Check apiVersion is supported
3. Validate all required fields present
4. Check commit hash format (40-char hex)
5. Validate timestamp format (ISO 8601)
6. Verify dependency graph consistency (requires/required_by)

**Mode: integrity**
1. For each dependency in lock file:
   - Check .graft/<dep-name>/ exists
   - Run `git rev-parse HEAD` in repository
   - Compare to commit hash in lock file
   - Report any mismatches

**Exit codes**:
- `0` - All validations passed
- `1` - Validation failed (invalid configuration)
- `2` - Integrity mismatch (lock vs .graft/)

**Output** (text):
```
✓ Config validation passed
  - graft.yaml is valid YAML
  - All dependencies have valid sources
  - All command references valid

✓ Lock file validation passed
  - graft.lock format is valid (apiVersion: graft/v0)
  - All required fields present
  - Dependency graph consistent

✓ Integrity verification passed
  - meta-kb: commit matches (abc123...)
  - standards-kb: commit matches (def456...)

All validations passed ✓
```

**Output** (errors):
```
✗ Config validation failed
  - Line 15: Invalid git URL 'not-a-url'
  - Line 23: Command 'migrate-v3' referenced but not defined

✗ Lock file validation failed
  - Dependency 'meta-kb': missing 'commit' field
  - Dependency 'standards-kb': invalid commit hash 'not-a-hash'

✗ Integrity verification failed
  - templates-kb: Expected abc123..., found def456...
    Run 'graft resolve' to sync

3 validation failures
```

**Output** (JSON):
```json
{
  "config": {
    "valid": false,
    "errors": [
      {
        "line": 15,
        "message": "Invalid git URL 'not-a-url'"
      }
    ]
  },
  "lock": {
    "valid": true,
    "errors": []
  },
  "integrity": {
    "valid": true,
    "mismatches": []
  },
  "overall": "failed"
}
```

**Validation Requirements**:

The implementation MUST:
- Support all three validation modes (config, lock, integrity)
- Return appropriate exit codes (0=success, 1=validation error, 2=integrity mismatch)
- Provide clear, actionable error messages
- Support both human-readable and JSON output formats

The implementation SHOULD:
- Report multiple errors, not just the first one
- Include line numbers for config errors where possible
- Suggest fixes for common errors

**Use Cases**:

1. **Pre-commit hook**:
```bash
#!/bin/bash
# .git/hooks/pre-commit
graft validate config lock
if [ $? -ne 0 ]; then
  echo "Graft validation failed. Fix errors before committing."
  exit 1
fi
```

2. **CI/CD pipeline**:
```yaml
# .github/workflows/validate.yml
- name: Validate Graft config
  run: graft validate --json
```

3. **Debug integrity issues**:
```bash
# Check if local .graft/ is in sync
graft validate integrity

# If mismatch, re-resolve
graft resolve
```

---

## Mutation Operations

### graft upgrade

**Purpose**: Upgrade a dependency to a new version. Atomic operation that runs migration, verification, and updates lock file.

**Syntax**:
```bash
graft upgrade <dep-name> [options]
```

**Options**:
- `--to <ref>`: Target ref (default: latest)
- `--dry-run`: Show what would be done without executing
- `--skip-migration`: Skip migration command (not recommended)
- `--skip-verify`: Skip verification command (not recommended)

**Behavior** (atomic):
1. Validate target ref exists
2. Create snapshot for rollback
3. Update files to new version
4. Run migration command (if defined)
5. Run verification command (if defined)
6. Update lock file
7. On failure: rollback all changes

**Output** (success):
```
Upgrading meta-kb: v1.5.0 → v2.0.0

Running migration: migrate-v2
  Command: npx jscodeshift -t codemods/v2.js src/
  ✓ Processed 15 files

Running verification: verify-v2
  Command: npm test && ! grep -r 'getUserData' src/
  ✓ 127 tests passed
  ✓ No occurrences of 'getUserData' found

✓ Upgrade complete
Updated graft.lock: meta-kb@v2.0.0
```

**Output** (failure):
```
Upgrading meta-kb: v1.5.0 → v2.0.0

Running migration: migrate-v2
  Command: npx jscodeshift -t codemods/v2.js src/
  ✓ Processed 15 files

Running verification: verify-v2
  Command: npm test && ! grep -r 'getUserData' src/
  ✗ 3 tests failed

Upgrade failed. Rolling back changes...
  ✓ Reverted file modifications

Lock file remains at v1.5.0

Error: Verification failed
To retry after fixing:
  1. Fix failing tests
  2. Run: graft upgrade meta-kb --to v2.0.0
```


---

### graft apply

**Purpose**: Update lock file to acknowledge a version without running migrations. For manual migration workflows.

**Syntax**:
```bash
graft apply <dep-name> --to <ref>
```

**Behavior**:
1. Validate ref exists
2. Update lock file immediately
3. Do not run migrations or verification

**Use case**: When user has manually performed migrations and wants to update lock file.

**Output**:
```
Applied meta-kb@v2.0.0
Updated graft.lock

Note: No migrations were run. Ensure you've completed all required migrations manually.
```


---

### graft validate

**Purpose**: Validate graft.yaml and graft.lock for correctness.

**Syntax**:
```bash
graft validate [options]
```

**Options**:
- `--schema`: Validate YAML schema only
- `--refs`: Validate git refs exist
- `--lock`: Validate lock file consistency

**Behavior**:
1. Validate graft.yaml structure
2. Check that all migration/verify commands exist
3. Check that all refs in changes exist in git
4. Validate lock file structure
5. Verify lock file refs resolve to expected commits

**Output** (success):
```
Validating graft.yaml...
  ✓ Schema is valid
  ✓ All migration commands exist
  ✓ All verify commands exist
  ✓ All refs exist in git repository

Validating graft.lock...
  ✓ Schema is valid
  ✓ All refs exist
  ✓ All commits match
  ✓ No integrity issues

Validation successful
```

**Output** (failure):
```
Validating graft.yaml...
  ✓ Schema is valid
  ✗ Change 'v2.0.0': migration 'migrate-v2' not found in commands
  ✗ Ref 'v3.0.0' does not exist in git repository

Validating graft.lock...
  ✓ Schema is valid
  ⚠ Warning: meta-kb ref 'main' has moved (commit changed)

Validation failed with 2 errors, 1 warning
```


---

## Command Execution

### graft <dep>:<command>

**Purpose**: Execute a command defined in dependency's graft.yaml.

**Syntax**:
```bash
graft <dep-name>:<command-name> [args...]
```

**Behavior**:
1. Load dependency's graft.yaml
2. Find command definition
3. Execute in consumer's context
4. Pass through stdout/stderr

**Example**:
```bash
$ graft meta-kb:migrate-v2

Running: npx jscodeshift -t codemods/v2.js src/
Processing 15 files...
✓ Completed
```


---

## Operation Flow Diagram

```
┌──────────────┐
│ graft status │ → Read lock file → Display current state
└──────────────┘

┌──────────────┐
│ graft fetch  │ → Fetch from remote → Update cache
└──────────────┘

┌───────────────┐
│ graft changes │ → Load graft.yaml → Filter & display
└───────────────┘

┌─────────────┐
│ graft show  │ → Load change details → Display
└─────────────┘

┌───────────────┐
│ graft upgrade │ → Create snapshot → Update files
└───────────────┘         ↓
                  Run migration
                          ↓
                  Run verification
                          ↓
                  Update lock file
                          ↓
                  (On fail: rollback)

┌──────────────┐
│ graft apply  │ → Update lock file (no migration)
└──────────────┘

┌─────────────────┐
│ graft validate  │ → Validate YAML → Check refs → Verify integrity
└─────────────────┘
```

## Related

- [Specification: Change Model](./change-model.md)
- [Specification: graft.yaml Format](./graft-yaml-format.md)
- [Specification: Lock File Format](./lock-file-format.md)
- [Decision 0004: Atomic Upgrades](../decisions/decision-0004-atomic-upgrades.md)
