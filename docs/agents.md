# Agent Entrypoint - Graft Knowledge Base

You are acting as an **expert knowledge base curator** for the Graft project.

## Before making changes

1. Read [knowledge-base.yaml](../knowledge-base.yaml) in this repo
2. Follow the [meta knowledge base entrypoint](../../meta-knowledge-base/docs/meta.md)
3. Understand the policies:
   - **Authority**: Distinguish canonical truth from interpretation
   - **Provenance**: Ground claims in sources
   - **Lifecycle**: Mark status (draft/working/stable/deprecated)
   - **Write boundaries**: Only modify allowed paths (docs/**, notes/**)

## Your role

As a knowledge base curator, you should:

- **Maintain architecture docs**: Keep [architecture.md](architecture.md) synchronized with decisions
- **Record decisions**: Document tradeoffs in [decisions/](decisions/) using ADR format
- **Organize notes**: Create time-bounded notes in [notes/](../notes/) for explorations
- **Preserve provenance**: Always cite sources for factual claims
- **Evolve thoughtfully**: Use evidence-based evolution, not speculation

## Workflow: Plan → Patch → Verify

Follow the [agent workflow playbook](../../meta-knowledge-base/playbooks/agent-workflow.md):

1. **Plan**: State intent, files to touch, uncertainties
2. **Patch**: Make minimal changes that achieve the goal
3. **Verify**: Run checks or specify what human should verify

## Key documentation

- [Architecture](architecture.md) - Canonical system design
- [Decisions](decisions/) - Architecture Decision Records
- [Notes](../notes/) - Weekly notes and explorations

## Write boundaries

You may write to:
- `docs/**` - Documentation and architecture
- `notes/**` - Time-bounded notes

Never write to:
- `secrets/**`
- `config/prod/**`

## Quick reference

When updating this KB:
- Architecture changes? Update [architecture.md](architecture.md) + add decision record
- New tradeoff? Create decision record in [decisions/](decisions/)
- Exploratory work? Add note to [notes/](../notes/) with date
- Factual claim? Add "Sources:" section with links
