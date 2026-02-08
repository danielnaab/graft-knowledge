---
status: draft
last-verified: 2026-02-08
owners: [human, agent]
---

# Workspace Configuration

## Intent

Define the workspace.yaml format that declares which repositories belong to a workspace and configures Grove's behavior. The workspace is Grove's core organizing primitive - a named collection of git repositories that you work with together.

## Non-goals

- **Not a complex project file** - Keep it simple, just repo declarations and basic settings
- **Not a replacement for graft.yaml** - Grove reads graft configs, doesn't duplicate them
- **Not environment-specific** - Same config works across machines (paths can be relative to home)
- **Not a build system** - Grove navigates and captures, doesn't build or deploy

## Behavior

### Basic Workspace Declaration

```gherkin
Given a workspace config at ~/.config/grove/workspace.yaml:
  """
  name: "my-project"
  repositories:
    - path: ~/src/graft-knowledge
    - path: ~/src/meta-knowledge-base
  """
When Grove launches
Then it loads both repositories into the workspace registry
And displays their names and status
```

### Repository with Tags

```gherkin
Given a repository declaration with tags:
  """
  repositories:
    - path: ~/finances
      tags: [finances, monthly-cadence, high-priority]
  """
When Grove displays the repository list
Then it shows the tags for filtering
And can sort by tag weights
```

### Capture Configuration

```gherkin
Given a workspace with capture config:
  """
  capture:
    inbox: ~/notes/inbox/
    auto_commit: true
  """
When a user captures a note
Then it is saved to ~/notes/inbox/YYYY-MM-DDTHH-MM-SS-title.md
And automatically committed with message "capture: <first line>"
```

### Capture Routing by Prefix

```gherkin
Given a workspace with capture routes:
  """
  capture:
    default_inbox: ~/notes/inbox/
    routes:
      - prefix: "@finances"
        path: ~/finances/notes/
      - prefix: "@todo"
        path: ~/notes/todo/
  """
When a user captures "@finances lunch $15"
Then the note is created at ~/finances/notes/YYYY-MM-DDTHH-MM-SS-finances-lunch-15.md
And the prefix is stripped from the filename
```

```gherkin
Given the same routing config
When a user captures "random thought" (no prefix)
Then the note is created at ~/notes/inbox/YYYY-MM-DDTHH-MM-SS-random-thought.md
```

### Repository Status Scripts

```gherkin
Given a repository with custom status checks:
  """
  repositories:
    - path: ~/finances
      status:
        - name: overdue
          run: |
            days=$(( ($(date +%s) - $(git log -1 --format=%ct)) / 86400 ))
            [ $days -gt 30 ] && echo "⚠️ Monthly close overdue ($days days)"
  """
When Grove refreshes status for the finances repository
Then it executes the status script in the repo's directory
And displays the output if the script succeeds with exit code 0
```

### Multiple Workspace Configurations

```gherkin
Given multiple workspace files exist:
  - ~/.config/grove/workspace-personal.yaml
  - ~/.config/grove/workspace-work.yaml
When user runs `grove --workspace work`
Then Grove loads workspace-work.yaml
```

```gherkin
Given the user launched Grove with workspace-personal yesterday
When user runs `grove` today (no workspace flag)
Then Grove loads workspace-personal.yaml (last used)
```

### Search Exclusions

```gherkin
Given a workspace with search config:
  """
  search:
    exclude: ["node_modules", ".git", "vendor", "target"]
  """
When Grove indexes the workspace for search
Then it skips all directories matching the exclusion patterns
```

## Format Specification

### Complete Schema

```yaml
# Workspace identity
name: string                    # Required: Workspace display name

# Repository declarations
repositories:                   # Required: At least one repository
  - path: string               # Required: Absolute or ~ path to git repo
    tags: [string]             # Optional: Labels for filtering/sorting
    status:                    # Optional: Custom status checks
      - name: string           # Required: Status check identifier
        run: string            # Required: Shell script (exit 0 = show output)

# Capture configuration
capture:                       # Optional: Quick capture settings
  inbox: string                # Default inbox location (path)
  default_inbox: string        # Alias for 'inbox'
  auto_commit: boolean         # Auto-commit captures (default: false)
  template: string             # Capture file template (multiline)
  routes:                      # Prefix-based routing
    - prefix: string           # Match prefix (e.g., "@finances")
      path: string             # Target directory
      template: string         # Optional: Override template

# Search configuration
search:                        # Optional: Search index settings
  exclude: [string]            # Directory patterns to skip

# Tag configuration
tag_weights:                   # Optional: Tag priority for sorting
  tag-name: number             # Higher numbers = higher priority
```

### Field Details

**name** (required)
- Purpose: Display name for the workspace
- Format: Any string
- Example: `"personal"`, `"grove-development"`

**repositories** (required, array)
- Purpose: List of git repositories in the workspace
- Minimum: 1 repository
- Each repository requires `path`, optional `tags` and `status`

**repositories[].path** (required)
- Purpose: Location of the git repository
- Format: Absolute path or `~`-prefixed path
- Must exist and be a git repository
- Example: `~/src/graft-knowledge`, `/absolute/path/to/repo`

**repositories[].tags** (optional, array)
- Purpose: Labels for filtering and organization
- Format: Array of strings, lowercase-with-hyphens convention
- Example: `[finances, high-priority, monthly-cadence]`

**repositories[].status** (optional, array)
- Purpose: Custom status checks for the repository
- Format: Array of status check objects
- Each check runs a shell script; exit 0 + output = displayed

**repositories[].status[].name** (required)
- Purpose: Identifier for the status check
- Format: String, kebab-case convention
- Example: `overdue`, `uncategorized`, `inbox-overflow`

**repositories[].status[].run** (required)
- Purpose: Shell script to determine status
- Format: Multiline string, executed with sh
- Convention: Exit 0 with output = show; Exit 0 no output = silent; Exit non-zero = error
- Script runs in repository's directory
- Environment: Standard shell + git available

**capture.inbox** or **capture.default_inbox**
- Purpose: Default location for captures without prefix routing
- Format: Absolute or `~`-prefixed directory path
- Must be writable
- Directory created if it doesn't exist

**capture.auto_commit** (optional, default: false)
- Purpose: Automatically commit captures to git
- Format: boolean
- Commit message format: `capture: <first line of note>`

**capture.template** (optional)
- Purpose: Template for capture file content
- Format: Multiline string with placeholders
- Placeholders: `{{date}}`, `{{content}}`
- Example:
  ```yaml
  template: |
    ---
    date: {{date}}
    ---
    {{content}}
  ```

**capture.routes** (optional, array)
- Purpose: Route captures to specific locations by prefix
- Format: Array of route objects
- Matching: First matching prefix wins
- Prefix stripped from filename after routing

**capture.routes[].prefix** (required)
- Purpose: Prefix to match (e.g., `@finances`)
- Format: String, typically starts with `@`
- Case-sensitive

**capture.routes[].path** (required)
- Purpose: Target directory for this prefix
- Format: Absolute or `~`-prefixed directory path

**capture.routes[].template** (optional)
- Purpose: Override workspace template for this route
- Format: Same as `capture.template`

**search.exclude** (optional, array)
- Purpose: Patterns to exclude from search indexing
- Format: Array of strings (directory names or glob patterns)
- Default: `[".git"]`
- Common additions: `node_modules`, `vendor`, `target`, `.next`

**tag_weights** (optional, object)
- Purpose: Define priority for tags (affects sorting)
- Format: Map of tag name to numeric weight
- Higher weight = higher priority in home page display
- Example: `high-priority: 100`, `monthly-cadence: 50`

## Edge Cases

### Missing Required Fields

```gherkin
Given a workspace config without 'name':
  """
  repositories:
    - path: ~/src/repo
  """
When Grove tries to load the config
Then it fails with error "workspace.yaml: missing required field 'name'"
```

### Non-existent Repository Path

```gherkin
Given a repository path that doesn't exist:
  """
  repositories:
    - path: ~/nonexistent
  """
When Grove loads the workspace
Then it shows a warning "Repository not found: ~/nonexistent"
And continues loading other repositories
```

### Repository Path is Not a Git Repo

```gherkin
Given a path that exists but isn't a git repository:
  """
  repositories:
    - path: ~/not-git
  """
When Grove loads the workspace
Then it shows a warning "Not a git repository: ~/not-git"
And continues loading other repositories
```

### Status Script Exits Non-Zero

```gherkin
Given a status check that fails:
  """
  status:
    - name: broken
      run: exit 1
  """
When Grove executes the status check
Then it logs the error internally
And does not display any status for that check
```

### Capture to Non-existent Directory

```gherkin
Given a capture inbox that doesn't exist:
  """
  capture:
    inbox: ~/notes/new-inbox/
  """
When a user captures a note
Then Grove creates the directory ~/notes/new-inbox/
And saves the capture file
```

### Overlapping Capture Prefixes

```gherkin
Given capture routes with overlapping prefixes:
  """
  routes:
    - prefix: "@fin"
      path: ~/fin/
    - prefix: "@finances"
      path: ~/finances/
  """
When a user captures "@finances note"
Then it matches the first prefix "@fin" (first match wins)
And routes to ~/fin/
```

**Resolution**: Document that first match wins, recommend specific prefixes first.

## Constraints

### Performance
- Config parse time < 10ms
- Status script execution timeout: 5 seconds per script
- Maximum 100 repositories per workspace (arbitrary limit for initial implementation)

### Security
- Status scripts run in repository directory with user's shell environment
- No elevation or special permissions
- Scripts inherit user's PATH and git credentials

### Compatibility
- YAML 1.2 format
- Paths support `~` expansion (home directory)
- Status scripts use `/bin/sh` (POSIX shell)
- Works across Linux, macOS, Windows (with sh available)

## Open Questions

- [ ] Should repos be auto-discovered (scan ~/src/) or explicit-only?
- [ ] Should status scripts have access to Grove's internal state?
- [ ] Should templates support more complex logic (conditionals, loops)?
- [ ] Should capture routing support regex patterns instead of just prefix matching?
- [ ] Should workspace configs support inheritance/composition (base + override)?
- [ ] Should status checks run in parallel or serially?
- [ ] Should there be a schema validation tool (grove validate-config)?

## Decisions

- **2026-02-07**: Status scripts are shell commands, not a DSL
  - Maximally flexible - can call any program
  - Composes with everything (Python, jq, database queries)
  - Zero learning curve (everyone knows shell)
  - Same pattern as graft commands

- **2026-02-07**: Workstreams are just multiple workspace files
  - Reuses workspace primitive with no modification
  - Switching workstreams = switching config files
  - No complex "active subset" logic needed
  - Launch with `grove --workspace <name>`

- **2026-02-07**: Tags are simple strings, not hierarchical
  - Flat is simpler than nested categories
  - Users can adopt their own conventions
  - Compose with filters and weights naturally

- **2026-02-07**: Capture routing uses prefix matching
  - Simple to understand and use
  - Works well with natural language ("`@finances` for money stuff")
  - First match wins (explicit ordering)

## Sources

- [Workspace UI Exploration (2026-02-06)](../../../notes/2026-02-06-workspace-ui-exploration.md) - Original workspace config design
- [Grove Workflow Hub Primitives (2026-02-07)](../../../notes/2026-02-07-grove-workflow-hub-primitives.md) - Six design primitives: status scripts, workstreams as configs, tags, capture routing
- [Status Check Syntax Exploration (2026-02-08)](../../../notes/2026-02-08-status-check-syntax-exploration.md) - Status script semantics and examples
