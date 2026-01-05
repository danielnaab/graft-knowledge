---
title: "Decision 0007: Mirror and Offline Support"
date: 2026-01-05
status: accepted
---

# Decision 0007: Mirror and Offline Support

## Context

Enterprise environments and security-conscious organizations often require:
- Internal mirrors of external git repositories
- Air-gapped or offline development environments
- Compliance with security policies that prohibit direct external access
- Reliability guarantees (internal mirrors always available)
- Faster access (local mirrors vs. internet)

Currently, Graft has no built-in mirror support. Users must manually modify source URLs in graft.yaml, which:
- Breaks reproducibility (different sources)
- Complicates collaboration (different team members use different URLs)
- Creates lock file conflicts (source field differs)

## Decision

**Add transparent mirror configuration to Graft.**

Mirrors allow URL pattern-based rewriting of git sources without modifying graft.yaml or graft.lock files.

### Configuration Format

Mirrors can be configured at two levels:

**Global** (`~/.graft/config.yaml`):
```yaml
mirrors:
  - pattern: "https://github.com/*"
    replace: "https://github-mirror.corp.internal/*"
    fallback: true  # Try original if mirror fails

  - pattern: "git@github.com:*"
    replace: "git@github-mirror.corp.internal:*"
    fallback: false  # Don't fallback (air-gapped environment)
```

**Project** (`graft.yaml`):
```yaml
apiVersion: graft/v0

# Project-specific mirrors (override global)
mirrors:
  - pattern: "https://github.com/myorg/*"
    replace: "https://internal-git.corp/mirrors/*"

deps:
  meta-kb: "https://github.com/myorg/meta-kb.git#v2.0.0"
```

### Semantics

1. **Pattern matching**: First matching pattern wins (top-to-bottom)
2. **Glob syntax**: `*` matches any characters within a path segment
3. **Rewriting**: Replace matched portion with replacement string
4. **Lock file transparency**: Original URL stored in lock file, not rewritten URL
5. **Fallback**: If `fallback: true`, try original URL if mirror fails
6. **Precedence**: Project mirrors override global mirrors

## Rationale

### Alignment with Design Principles

**1. Git-Native** ✅
- Mirrors work with git's existing capabilities
- No new version control concepts
- Leverages git's native mirroring/cloning

**2. Explicit Over Implicit** ✅
- Mirrors must be explicitly configured
- Clear configuration files (not environment variables)
- Documented and validatable

**3. Minimal Primitives** ✅
- No new core concepts (just URL rewriting)
- Simple pattern matching, no complex logic
- Builds on existing git infrastructure

**4. Separation of Concerns** ✅
- Mirror configuration separate from dependency declaration
- graft.yaml declares dependencies (original URLs)
- Config files handle environment-specific routing

**5. Reproducibility** ✅
- Lock file stores original URLs
- Mirrors are transparent transformation
- Same lock file works across environments (with different mirrors)

### Real-World Use Cases

**Use Case 1: Corporate Environment**

```yaml
# Global config for all developers in company
mirrors:
  - pattern: "https://github.com/*"
    replace: "https://github.corp-mirror.internal/*"
    fallback: true
```

- All external dependencies automatically mirrored
- Developers don't modify graft.yaml
- Lock files remain clean with original URLs
- Works from office or home (with fallback)

**Use Case 2: Air-Gapped Environment**

```yaml
# No fallback - must use internal mirrors only
mirrors:
  - pattern: "https://github.com/*"
    replace: "https://internal-git.corp/mirrors/github/*"
    fallback: false

  - pattern: "git@github.com:*"
    replace: "git@internal-git.corp:mirrors/github/*"
    fallback: false
```

- Ensures no external access
- Clear failure if mirror unavailable
- Compliance with security policies

**Use Case 3: Faster Local Development**

```yaml
# Use local network mirror for speed
mirrors:
  - pattern: "https://github.com/myorg/*"
    replace: "file:///srv/git-mirrors/myorg/*"
    fallback: true
```

- Instant clones from local filesystem
- Fallback to github.com if mirror stale

**Use Case 4: Fork Substitution**

```yaml
# Use fork for specific dependency
mirrors:
  - pattern: "https://github.com/upstream/old-kb.git"
    replace: "https://github.com/myorg/old-kb-fork.git"
    fallback: false
```

- Temporary patch while waiting for upstream
- Project-specific override in graft.yaml

### Comparison to Other Tools

**npm** - Registry mirrors:
```
npm config set registry https://registry.npm.corp.internal/
```
- Global registry replacement
- Similar concept but package-specific

**cargo** - Source replacement:
```toml
[source.crates-io]
replace-with = "internal-mirror"

[source.internal-mirror]
registry = "https://crates.corp.internal"
```
- More complex but similar pattern matching

**pip** - Index URL override:
```
pip install --index-url https://pypi.corp.internal/simple
```
- Command-line flag approach

**git** - URL insteadOf:
```
git config url."https://mirror.corp/".insteadOf "https://github.com/"
```
- Git-level rewriting (Graft mirrors inspired by this)

**Assessment**: Graft's approach aligns with industry practices while being simpler and more explicit.

## Alternatives Considered

### Alternative 1: Git Config Integration

Use git's built-in `url.insteadOf` configuration:

```bash
git config --global url."https://mirror.corp/".insteadOf "https://github.com/"
```

**Rejected because**:
- Not Graft-specific (affects all git operations)
- Harder to validate and document
- No fallback mechanism
- Less explicit (hidden in git config)

### Alternative 2: Environment Variables

```bash
export GRAFT_MIRROR_GITHUB="https://mirror.corp/"
graft resolve
```

**Rejected because**:
- Less explicit than configuration files
- Not easily validated or versioned
- Harder to document and discover
- Environment variables are implicit

### Alternative 3: Rewrite Lock File URLs

Store mirrored URLs in lock file:

```yaml
dependencies:
  meta-kb:
    source: "https://mirror.corp/meta-kb.git"  # Rewritten
```

**Rejected because**:
- Breaks reproducibility (different environments have different lock files)
- Git merge conflicts when team members use different mirrors
- Original source information lost
- Violates transparency principle

### Alternative 4: No Mirror Support

Document manual workarounds (forking, hand-editing URLs).

**Rejected because**:
- Significant friction for enterprise users
- Common requirement across industries
- Reduces adoption in corporate settings
- Other tools provide this (competitive disadvantage)

## Consequences

### Positive

- **Enterprise-friendly**: Meets corporate security and reliability requirements
- **Zero friction**: Works transparently without modifying dependencies
- **Reproducible**: Lock files remain clean with original URLs
- **Flexible**: Supports multiple use cases (mirrors, forks, local dev)
- **Simple**: Just URL pattern matching, no complex logic

### Negative

- **Configuration complexity**: New concept to learn
- **Debug difficulty**: Mirror failures might be non-obvious
  - *Mitigated by*: Clear error messages showing attempted URLs
- **Documentation burden**: Must explain mirrors and patterns

### Neutral

- **Optional feature**: Projects without mirrors work unchanged
- **Incremental adoption**: Can add mirrors later when needed

## Implementation

### Configuration File Location

**Global**: `~/.graft/config.yaml` or `$XDG_CONFIG_HOME/graft/config.yaml`

**Project**: `graft.yaml` (mirrors section)

### Mirror Matching Algorithm

```python
def resolve_mirror(original_url: str, mirrors: list[Mirror]) -> str:
    """
    Resolve mirror for given URL.
    Returns mirrored URL if pattern matches, else original.
    """
    for mirror in mirrors:
        if matches_pattern(original_url, mirror.pattern):
            return apply_replacement(original_url, mirror.pattern, mirror.replace)

    return original_url  # No mirror matched

def clone_with_fallback(url: str, mirror: Mirror | None) -> None:
    """
    Clone from mirror with optional fallback.
    """
    if mirror:
        try:
            git_clone(mirror.url)
        except GitError as e:
            if mirror.fallback:
                print(f"Mirror failed: {e}. Trying original URL...")
                git_clone(url)
            else:
                raise MirrorError(f"Mirror {mirror.url} failed: {e}")
    else:
        git_clone(url)
```

### Error Messages

```bash
# Mirror fails, no fallback
Error: Failed to clone dependency 'meta-kb'
  Original URL: https://github.com/org/meta-kb.git
  Mirror URL:   https://mirror.corp/org/meta-kb.git (FAILED)
  Error: Repository not found

  Mirror configuration:
    Pattern: https://github.com/*
    Replace: https://mirror.corp/*
    Fallback: disabled

  Suggestions:
  - Check mirror is up-to-date
  - Verify mirror URL is correct
  - Enable fallback if appropriate

# Mirror fails, fallback succeeds
Warning: Mirror failed, using original URL
  Original URL: https://github.com/org/meta-kb.git
  Mirror URL:   https://mirror.corp/org/meta-kb.git (FAILED)
  Fallback URL: https://github.com/org/meta-kb.git (SUCCESS)
```

### Validation

```bash
# Validate mirror configuration
graft validate --mirrors

# Check mirrors:
✓ Pattern 'https://github.com/*' is valid
✓ Replacement 'https://mirror.corp/*' is valid
✓ Test rewrite: https://github.com/org/repo.git
  → https://mirror.corp/org/repo.git

# Warn about issues:
⚠ Mirror 'https://mirror.corp' is not reachable
  Consider enabling fallback
```

## Migration Path

### Phase 1: Add Basic Mirror Support

- Parse mirrors from config files
- Implement pattern matching
- Implement URL rewriting
- Basic fallback support

### Phase 2: Enhanced Error Handling

- Detailed error messages
- Validation command
- Mirror testing utilities

### Phase 3: Advanced Features (Optional)

- Regex patterns (if glob insufficient)
- Mirror priorities/ordering
- Mirror health checking
- Caching hints

## Future Enhancements

If needed, consider:

1. **Authenticated mirrors**: Credential handling for private mirrors
2. **Mirror discovery**: Auto-detect mirrors via DNS or well-known URLs
3. **Mirror verification**: Ensure mirror content matches upstream
4. **Performance hints**: Mark mirrors as "fast" for prioritization

## Related

- [graft.yaml Format Specification](../specification/graft-yaml-format.md) - Will need mirrors section
- [Decision 0002: Git Refs Over Semver](./decision-0002-git-refs-over-semver.md) - Git-native approach
- [Lock File Format Specification](../specification/lock-file-format.md) - Stores original URLs

## References

- **Git URL rewriting**: https://git-scm.com/docs/git-config#Documentation/git-config.txt-urlltbasegtinsteadOf
- **npm registry configuration**: https://docs.npmjs.com/cli/v9/using-npm/config#registry
- **Cargo source replacement**: https://doc.rust-lang.org/cargo/reference/source-replacement.html
