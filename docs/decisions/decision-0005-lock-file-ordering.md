---
title: "Decision 0005: Lock File Ordering Conventions"
date: 2026-01-05
status: accepted
---

# Decision 0005: Lock File Ordering Conventions

## Context

The `graft.lock` file contains all resolved dependencies (both direct and transitive). Without a specified ordering, different tool implementations might generate lock files with dependencies in different orders, leading to:

- Noisy git diffs when regenerating lock files
- Reduced human readability
- Inconsistent behavior across tools

## Decision

Lock files **SHOULD** order dependencies as follows:

1. **Direct dependencies first** - All entries with `direct: true`
2. **Transitive dependencies second** - All entries with `direct: false`
3. **Alphabetically within each group** - Sort by dependency name within direct and transitive groups

However, **parsers MUST accept dependencies in any order** to allow:
- Hand-editing when necessary
- Backward compatibility
- Tool flexibility

## Rationale

### Benefits of Conventional Ordering

**1. Improved Readability**
```yaml
# Easy to scan - direct deps at top
dependencies:
  meta-kb: {...}        # Direct - user declared
  docs-kb: {...}        # Direct - user declared
  standards-kb: {...}   # Transitive - pulled in
  templates-kb: {...}   # Transitive - pulled in
```

**2. Meaningful Git Diffs**

When dependencies change, diffs show relevant changes:
```diff
 dependencies:
   meta-kb:
     ref: "v2.0.0"
+    requires: ["standards-kb", "utils-kb"]  # New transitive dep

+  utils-kb:  # Added transitive dependency
+    source: "..."
```

**3. Consistency Across Tools**

All Graft implementations generate lock files in the same order, reducing churn and confusion.

### Why Not Enforce Strict Ordering?

**Flexibility**: Users may need to hand-edit lock files in emergencies. Requiring strict order would make this harder.

**Robustness**: Tools should be resilient to different orderings. The lock file is a data structure, not a pretty-printed format.

**Principle**: "Be strict in what you generate, liberal in what you accept"

## Alternatives Considered

### Alternative 1: Alphabetical Only

Order all dependencies alphabetically without grouping direct/transitive.

**Rejected because**:
- Loses important semantic distinction
- Harder to identify which deps are declared vs. pulled in
- Less informative at a glance

### Alternative 2: Dependency Tree Order

Order dependencies to reflect tree structure (parent before children).

```yaml
dependencies:
  meta-kb: {...}
    standards-kb: {...}  # meta-kb's dep
      templates-kb: {...}  # standards-kb's dep
```

**Rejected because**:
- YAML doesn't have indentation semantics for objects
- Shared dependencies break tree visualization
- Complexity outweighs benefits

### Alternative 3: Insertion Order

Preserve order in which dependencies were resolved.

**Rejected because**:
- Non-deterministic (depends on resolution algorithm)
- Different tools might use different traversal orders
- No semantic meaning

### Alternative 4: Strict Enforcement

Require parsers to reject incorrectly ordered lock files.

**Rejected because**:
- Too rigid - prevents hand-editing
- Doesn't add safety value
- Violates "liberal in what you accept" principle

## Consequences

### Positive

- **Consistency**: All tools generate same order
- **Readability**: Users can quickly scan direct vs. transitive deps
- **Git-friendly**: Diffs are cleaner and more meaningful
- **Flexible**: Hand-editing still allowed

### Negative

- **Documentation burden**: Tools must document the convention
- **Implementation**: Generators must sort dependencies
- **Potential confusion**: Users might expect enforcement

### Neutral

- **Non-normative**: It's a SHOULD, not a MUST
- **No breaking changes**: Existing lock files remain valid

## Implementation

### For Lock File Generators

```python
def write_lock_file(resolved_deps: dict[str, Dependency]) -> None:
    """Generate graft.lock with conventional ordering."""
    lock_data = {"apiVersion": "graft/v0", "dependencies": {}}

    # Separate direct and transitive
    direct = {k: v for k, v in resolved_deps.items() if v.direct}
    transitive = {k: v for k, v in resolved_deps.items() if not v.direct}

    # Write direct deps first (alphabetically)
    for name in sorted(direct.keys()):
        lock_data["dependencies"][name] = serialize(direct[name])

    # Then transitive deps (alphabetically)
    for name in sorted(transitive.keys()):
        lock_data["dependencies"][name] = serialize(transitive[name])

    write_yaml("graft.lock", lock_data)
```

### For Lock File Parsers

```python
def read_lock_file(path: str) -> dict[str, Dependency]:
    """Parse graft.lock, accepting any dependency order."""
    data = yaml.safe_load(open(path))

    # No order validation - accept as-is
    return {
        name: parse_dependency(dep_data)
        for name, dep_data in data["dependencies"].items()
    }
```

## Related

- [Lock File Format Specification](../specification/lock-file-format.md)
- [Decision 0003: Explicit Change Declarations](./decision-0003-explicit-change-declarations.md) - Similar philosophy of explicit structure

## References

- **RFC 2119** - Key words MUST, SHOULD, MAY: https://www.ietf.org/rfc/rfc2119.txt
- **Robustness Principle** - "Be conservative in what you send, liberal in what you accept"
- **YAML Specification** - YAML maps are unordered: https://yaml.org/spec/1.2/spec.html
