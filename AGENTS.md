# OpenShift Pipelines Architecture - Agent Guide

This repository hosts Architecture Decision Records (ADRs) for OpenShift Pipelines.

## Repository Conventions

1. **Scope**: Only product-level downstream decisions belong here (OpenShift integration, packaging, security/compliance, Konflux CI/CD, downstream patches). Upstream Tekton features belong in [TEPs](https://github.com/tektoncd/community/tree/main/teps).
2. **Confidentiality**: Never include customer names or private cluster details.
3. **Template**: All ADRs are stored under `ADR/00XX-<slug>.md` and must follow `ADR/0000-adr-template.md`.

## Available Skills & Tools

- **ADR Skill**: `skills/adr/SKILL.md` — Guide for grilling, evaluating trade-offs, and authoring ADRs.
- **Slash Command**: `/adr` (see `commands/adr.md`).

## Quick ADR Workflow for Agents

When a user asks to write or discuss an ADR:
1. Load `skills/adr/SKILL.md`.
2. Inspect existing ADRs under `ADR/` to determine the next sequential ID and check for related prior decisions.
3. Walk the user through the decision tree (one question at a time) before generating markdown.
4. Output the completed or work-in-progress ADR using standard YAML frontmatter and template sections (ADRs can be merged as `status: proposed` and refined iteratively).
