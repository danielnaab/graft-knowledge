---
title: "Dependency Update Notification Specification"
date: 2026-01-05
status: draft
---

# Dependency Update Notification Specification

## Overview

This specification defines how Graft-aware repositories can automatically detect and propagate dependency updates. When an upstream dependency is updated, consuming repositories should be notified and offered the opportunity to update.

The goal is **zero-configuration for upstream repositories**—the dependency graph is already declared in `graft.yaml` files; this specification describes how to leverage that existing information for automated update propagation.

## Requirements

### Functional Requirements

1. **Automatic detection**: Consumers must be able to detect when upstream dependencies have new commits beyond their pinned ref
2. **PR creation**: When updates are detected, the system should create pull requests in consumer repositories with updated refs
3. **Workspace continuity**: PRs should include context for continuation if implementation work is needed (e.g., links to task workspaces)
4. **No upstream configuration**: Upstream repositories should not need to maintain lists of their consumers

### Non-Functional Requirements

1. **Fast execution**: Update checks should complete quickly without heavy build steps
2. **Minimal dependencies**: Implementation should rely on standard tools (shell, git, curl)
3. **Idempotent**: Running the check multiple times should not create duplicate PRs
4. **Extensible**: Easy to add new consumer repositories without central coordination

## Core Concepts

### Dependency Graph Inversion

The `graft.yaml` files in consumer repositories already declare the dependency graph:

```yaml
# In consumer repo (e.g., graft)
apiVersion: graft/v0
deps:
  graft-knowledge: "ssh://forgejo@platform-vm:2222/daniel/graft-knowledge.git#main"
```

To find all consumers of a given upstream:
1. Enumerate all repositories in the organization
2. Parse each repository's `graft.yaml`
3. Build inverted index: `upstream → [consumers]`

This approach requires no configuration in upstream repositories.

### Update Detection

An update is available when:
```
upstream_head != consumer_pinned_ref
```

Where:
- `upstream_head`: Current HEAD of the upstream repository's tracked branch
- `consumer_pinned_ref`: The ref (commit, tag, branch) specified in the consumer's `graft.yaml`

Detection uses `git ls-remote` for efficiency (no clone required):
```bash
git ls-remote <upstream-url> HEAD
```

### PR Content

Update PRs should include:

1. **Updated `graft.yaml`**: New ref pointing to upstream HEAD
2. **Summary**: Which dependency was updated and to what ref
3. **Upstream context**: Commit message(s) from upstream
4. **Continuation link**: URL to resume work if manual intervention needed

## Event Strategy

See [Decision 0006: Dependency Update Event Strategy](../decisions/decision-0006-dependency-update-events.md) for the rationale behind the chosen approach.

### Recommended: Organization-wide Event with Polling Fallback

1. **Primary trigger**: Organization-level webhook fires on any repository push
2. **Handler**: Central workflow receives event, identifies affected consumers, creates PRs
3. **Fallback**: Scheduled polling (e.g., hourly) catches any missed events

This provides near-real-time updates without requiring upstream repositories to know about their consumers.

### Alternative Strategies

Other valid strategies (see ADR for full analysis):

- **Pure polling**: Scheduled checks without webhooks (simpler, higher latency)
- **Per-consumer workflows**: Each consumer runs its own update check (decentralized)
- **Upstream notification**: Upstream maintains consumer list (rejected—violates zero-config requirement)

## Implementation Components

### Central Update Service

A dedicated repository (e.g., `graft-ci`) containing:

1. **Update check workflow**: Triggered by org events or schedule
2. **Dependency graph builder**: Scans org repos, parses `graft.yaml` files
3. **PR creator**: Creates/updates PRs in consumer repositories

### Workflow Inputs

When triggered by push event:
```yaml
inputs:
  pushed_repo: string    # Repository that was updated
  pushed_ref: string     # Branch/ref that was pushed to
  pushed_sha: string     # New HEAD commit
```

When triggered by schedule:
- No inputs; scan all repos for any outdated dependencies

### PR Creation

```yaml
# Pseudocode for PR content
title: "chore(deps): update {dep_name} to {short_sha}"

body: |
  ## Summary
  Updates `{dep_name}` dependency to latest upstream commit.

  **Upstream commit:** `{sha}` - {commit_message}
  **Author:** {author}

  ## Continuation
  {workspace_link if available}

  ---
  Automated by graft-ci
```

## Integration with graft.yaml

### Current Format Support

The specification works with the existing `graft.yaml` format:

```yaml
apiVersion: graft/v0
deps:
  dep-name: "git-url#ref"
```

The `#ref` portion is updated to the new commit SHA.

### Extended Format (Future)

Future versions may support additional metadata:

```yaml
apiVersion: graft/v1
deps:
  dep-name:
    source: "git-url"
    ref: "main"
    auto_update: true        # Opt-in to automatic updates
    update_strategy: "pr"    # pr | auto-merge | notify-only
```

## CLI Integration

### graft check-updates

Query operation to check for available updates:

```bash
graft check-updates [options]

Options:
  --json           Output as JSON
  --create-pr      Create PR if updates available (requires git remote access)
  --dry-run        Show what would be done without making changes
```

**Output:**
```
Checking dependencies...
  graft-knowledge: abc123 → def456 (3 commits behind)
  meta-kb: up to date

1 update available. Run with --create-pr to create pull request.
```

### graft update

Mutation operation to update dependencies locally:

```bash
graft update [dep-name] [options]

Options:
  --all            Update all dependencies
  --to <ref>       Update to specific ref (default: upstream HEAD)
```

## Security Considerations

1. **Write access**: Central service needs write access to consumer repositories
2. **Credential management**: Use deploy keys or fine-grained tokens with minimal scope
3. **Branch protection**: PRs should go through normal review process, not direct push
4. **Webhook validation**: Verify webhook signatures to prevent spoofing

## Related

- [Decision 0006: Dependency Update Event Strategy](../decisions/decision-0006-dependency-update-events.md)
- [Specification: graft.yaml Format](./graft-yaml-format.md)
- [Specification: Core Operations](./core-operations.md)

## References

- Forgejo Actions documentation: https://forgejo.org/docs/latest/user/actions/
- Git ls-remote: https://git-scm.com/docs/git-ls-remote
