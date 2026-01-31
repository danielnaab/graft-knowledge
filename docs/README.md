# Graft Project Knowledge Base

This KB follows the [meta knowledge base](../../meta-knowledge-base/docs/meta.md) system.

## Overview

Graft is a task runner and git-centered package manager that aims to simplify:
- Configurable task execution
- Git-based dependency management
- Reproducible development workflows

## Documentation Structure

- **[Specifications](specification/)** - Formal specifications for implementers
  - [Lock File Format](specification/lock-file-format.md) - State tracking format
  - [graft.yaml Format](specification/graft-yaml-format.md) - Configuration file format
  - [Core Operations](specification/core-operations.md) - Operation semantics and behavior
  - [Change Model](specification/change-model.md) - Data model for changes
  - [Dependency Layout](specification/dependency-layout.md) - How dependencies are organized
  - [Dependency Update Notification](specification/dependency-update-notification.md) - Automated update propagation

- **[Decisions](decisions/)** - Architecture Decision Records (ADRs)
  - Documents key architectural choices with rationale
  - Captures alternatives considered and trade-offs

- **[Architecture](architecture.md)** - System design and core concepts overview

- **[Use Cases](use-cases.md)** - What Graft enables and why

- **[CHANGELOG](../CHANGELOG.md)** - Track specification changes and additions

- **[Notes](../notes/)** - Working notes, brainstorming, and design analysis

## For Implementers

See [CHANGELOG](../CHANGELOG.md) for recent specification changes.

Implement against a specific version by referencing the git commit or tag.
See CHANGELOG for guidance on pinning implementations to specifications.
