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

**Implementation**:
```python
def status(dep_name: Optional[str] = None, check_updates: bool = False) -> dict:
    """Show dependency status."""
    lock = read_lock_file()
    results = {}

    deps = [dep_name] if dep_name else lock['dependencies'].keys()

    for dep in deps:
        dep_data = lock['dependencies'][dep]
        current_ref = dep_data['ref']

        result = {
            'current': current_ref,
            'consumed_at': dep_data['consumed_at'],
            'available': None,
            'pending_changes': 0
        }

        if check_updates:
            # Fetch latest from upstream
            latest_ref = get_latest_ref(dep)
            if latest_ref != current_ref:
                result['available'] = latest_ref
                result['pending_changes'] = count_changes_between(
                    dep, current_ref, latest_ref
                )

        results[dep] = result

    return results
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

**Implementation**:
```python
def changes(
    dep_name: str,
    from_ref: Optional[str] = None,
    to_ref: Optional[str] = None,
    change_type: Optional[str] = None,
    breaking_only: bool = False
) -> list[Change]:
    """List changes for a dependency."""

    # Get current consumed version if not specified
    if from_ref is None:
        lock = read_lock_file()
        from_ref = lock['dependencies'][dep_name]['ref']

    # Get latest version if not specified
    if to_ref is None:
        to_ref = get_latest_ref(dep_name)

    # Load changes from dependency's graft.yaml
    all_changes = load_changes(dep_name)

    # Filter to range
    changes_in_range = filter_changes_between(all_changes, from_ref, to_ref)

    # Filter by type
    if breaking_only:
        changes_in_range = [c for c in changes_in_range if c.type == 'breaking']
    elif change_type:
        changes_in_range = [c for c in changes_in_range if c.type == change_type]

    return changes_in_range
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

**Implementation**:
```python
def show(dep_name: str, ref: str) -> dict:
    """Show details of a specific change."""
    config = load_graft_yaml(dep_name)

    # Get change
    change_data = config.get('changes', {}).get(ref)
    if not change_data:
        raise ValueError(f"Change {ref} not found for {dep_name}")

    change = Change(ref=ref, **change_data)

    # Get command details
    result = {
        'ref': change.ref,
        'type': change.type,
        'description': change.description,
        'migration': change.migration,
        'verify': change.verify
    }

    # Add command details
    commands = config.get('commands', {})
    if change.migration and change.migration in commands:
        result['migration_command'] = commands[change.migration]

    if change.verify and change.verify in commands:
        result['verify_command'] = commands[change.verify]

    return result
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

**Implementation**:
```python
def fetch(dep_name: Optional[str] = None):
    """Fetch latest from upstream."""
    lock = read_lock_file()

    deps = [dep_name] if dep_name else lock['dependencies'].keys()

    for dep in deps:
        source = lock['dependencies'][dep]['source']

        # Fetch from git remote
        subprocess.run(['git', 'fetch'], cwd=get_dep_cache_path(dep))

        # Update local metadata
        update_dep_cache(dep)

        print(f"✓ Fetched {dep}")
```

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
5. Validate mirror patterns (if present)

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

**Implementation**:
```python
def validate(mode: str = "all", json_output: bool = False) -> int:
    """
    Validate graft configuration and state.
    Returns exit code.
    """
    results = {}

    if mode in ["all", "config"]:
        results["config"] = validate_config()

    if mode in ["all", "lock"]:
        results["lock"] = validate_lock_file()

    if mode in ["all", "integrity"]:
        results["integrity"] = validate_integrity()

    if json_output:
        print(json.dumps(results))
    else:
        print_validation_results(results)

    # Determine exit code
    if any(not r["valid"] for r in results.values()):
        return 1 if "config" in results or "lock" in results else 2

    return 0

def validate_config() -> dict:
    """Validate graft.yaml."""
    errors = []

    try:
        config = yaml.safe_load(open("graft.yaml"))
    except yaml.YAMLError as e:
        return {"valid": False, "errors": [{"message": f"Invalid YAML: {e}"}]}

    # Check dependencies
    if "dependencies" in config:
        for name, dep in config["dependencies"].items():
            if "source" not in dep:
                errors.append({"message": f"Dependency '{name}': missing 'source'"})
            elif not is_valid_git_url(dep["source"]):
                errors.append({"message": f"Dependency '{name}': invalid git URL"})

    # Check commands
    if "commands" in config:
        for cmd_name, cmd in config["commands"].items():
            if "run" not in cmd:
                errors.append({"message": f"Command '{cmd_name}': missing 'run'"})

    # Check mirrors
    if "mirrors" in config:
        for i, mirror in enumerate(config["mirrors"]):
            if "pattern" not in mirror:
                errors.append({"message": f"Mirror {i}: missing 'pattern'"})
            if "replace" not in mirror:
                errors.append({"message": f"Mirror {i}: missing 'replace'"})

    return {"valid": len(errors) == 0, "errors": errors}

def validate_lock_file() -> dict:
    """Validate graft.lock."""
    errors = []

    try:
        lock = yaml.safe_load(open("graft.lock"))
    except FileNotFoundError:
        return {"valid": False, "errors": [{"message": "graft.lock not found"}]}
    except yaml.YAMLError as e:
        return {"valid": False, "errors": [{"message": f"Invalid YAML: {e}"}]}

    # Check apiVersion
    if "apiVersion" not in lock:
        errors.append({"message": "Missing 'apiVersion' field"})
    elif lock["apiVersion"] not in ["graft/v0", "graft/v1"]:
        errors.append({"message": f"Unsupported apiVersion: {lock['apiVersion']}"})

    # Check dependencies
    if "dependencies" not in lock:
        errors.append({"message": "Missing 'dependencies' section"})
    else:
        for name, dep in lock["dependencies"].items():
            # Required fields
            for field in ["source", "ref", "commit", "consumed_at", "direct", "requires", "required_by"]:
                if field not in dep:
                    errors.append({"message": f"Dependency '{name}': missing '{field}'"})

            # Validate commit hash
            if "commit" in dep and not re.match(r"^[0-9a-f]{40}$", dep["commit"]):
                errors.append({"message": f"Dependency '{name}': invalid commit hash"})

            # Validate timestamp
            if "consumed_at" in dep:
                try:
                    datetime.fromisoformat(dep["consumed_at"].replace("Z", "+00:00"))
                except ValueError:
                    errors.append({"message": f"Dependency '{name}': invalid timestamp"})

    return {"valid": len(errors) == 0, "errors": errors}

def validate_integrity() -> dict:
    """Validate .graft/ directory matches lock file."""
    mismatches = []

    lock = yaml.safe_load(open("graft.lock"))

    for name, dep in lock.get("dependencies", {}).items():
        dep_path = f".graft/{name}"

        if not os.path.exists(dep_path):
            mismatches.append({
                "dependency": name,
                "error": "Directory not found",
                "expected": dep["commit"]
            })
            continue

        # Get actual commit
        result = subprocess.run(
            ["git", "rev-parse", "HEAD"],
            cwd=dep_path,
            capture_output=True,
            text=True
        )

        if result.returncode != 0:
            mismatches.append({
                "dependency": name,
                "error": "Not a git repository"
            })
            continue

        actual_commit = result.stdout.strip()
        expected_commit = dep["commit"]

        if actual_commit != expected_commit:
            mismatches.append({
                "dependency": name,
                "expected": expected_commit,
                "actual": actual_commit
            })

    return {"valid": len(mismatches) == 0, "mismatches": mismatches}
```

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

See: [Recommendation #2: Add graft validate Command](../../graft/docs/plans/graft-improvements-recommendations.md#recommendation-2-add-graft-validate-command)

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

**Implementation**:
```python
def upgrade(
    dep_name: str,
    to_ref: str,
    dry_run: bool = False,
    skip_migration: bool = False,
    skip_verify: bool = False
) -> bool:
    """Upgrade dependency (atomic operation)."""

    if dry_run:
        return preview_upgrade(dep_name, to_ref)

    # Get change details
    change = get_change(dep_name, to_ref)

    # Create snapshot for rollback
    snapshot = create_snapshot()

    try:
        # Step 1: Update files
        update_dependency_files(dep_name, to_ref)

        # Step 2: Run migration
        if change.migration and not skip_migration:
            execute_command(dep_name, change.migration)

        # Step 3: Run verification
        if change.verify and not skip_verify:
            execute_command(dep_name, change.verify)

        # Step 4: Update lock file
        update_lock_file(
            dep_name,
            ref=to_ref,
            commit=resolve_ref_to_commit(dep_name, to_ref),
            source=get_dep_source(dep_name)
        )

        print(f"✓ Upgrade complete: {dep_name}@{to_ref}")
        return True

    except Exception as e:
        # Rollback on any failure
        print(f"Upgrade failed: {e}")
        print("Rolling back changes...")
        restore_snapshot(snapshot)
        print(f"Lock file remains at {get_consumed_ref(dep_name)}")
        return False
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

**Implementation**:
```python
def apply(dep_name: str, to_ref: str):
    """Apply version without running migrations."""

    # Validate ref exists
    validate_ref_exists(dep_name, to_ref)

    # Update lock file
    update_lock_file(
        dep_name,
        ref=to_ref,
        commit=resolve_ref_to_commit(dep_name, to_ref),
        source=get_dep_source(dep_name)
    )

    print(f"Applied {dep_name}@{to_ref}")
    print("Updated graft.lock")
    print("\nNote: No migrations were run. Ensure you've completed all required migrations manually.")
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

**Implementation**:
```python
def validate(schema_only: bool = False, refs_only: bool = False, lock_only: bool = False):
    """Validate configuration."""
    errors = []
    warnings = []

    if not lock_only:
        # Validate graft.yaml
        config = load_graft_yaml()
        errors.extend(validate_graft_yaml(config))

        if not schema_only:
            errors.extend(validate_refs_exist(config))

    if not schema_only and not refs_only:
        # Validate graft.lock
        lock = read_lock_file()
        errors.extend(validate_lock_file(lock))
        warnings.extend(check_lock_integrity(lock))

    # Report results
    if errors:
        print(f"Validation failed with {len(errors)} errors")
        for error in errors:
            print(f"  ✗ {error}")
        return False
    elif warnings:
        print(f"Validation passed with {len(warnings)} warnings")
        for warning in warnings:
            print(f"  ⚠ {warning}")
        return True
    else:
        print("✓ Validation successful")
        return True
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

**Implementation**:
```python
def execute_command(dep_name: str, command_name: str, args: list[str] = None):
    """Execute a command from dependency's graft.yaml."""

    # Load command definition
    config = load_graft_yaml(dep_name)
    commands = config.get('commands', {})

    if command_name not in commands:
        raise ValueError(f"Command '{command_name}' not found in {dep_name}/graft.yaml")

    cmd_def = commands[command_name]
    cmd = cmd_def['run']

    # Execute
    working_dir = cmd_def.get('working_dir', '.')
    env = os.environ.copy()
    env.update(cmd_def.get('env', {}))

    # Append args if provided
    if args:
        cmd = f"{cmd} {' '.join(args)}"

    result = subprocess.run(
        cmd,
        shell=True,
        cwd=working_dir,
        env=env,
        capture_output=False  # Stream to stdout/stderr
    )

    if result.returncode != 0:
        raise RuntimeError(f"Command failed with exit code {result.returncode}")
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
