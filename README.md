# OpenShift Pipelines Architecture

Architecture Decision Records (ADRs) and documentation for OpenShift Pipelines.

## Why this repository?

OpenShift Pipelines makes many product-level decisions that aren't captured anywhere. This repository exists to:

- **Help newcomers** understand what's happening, why we do things a certain way, and how decisions were made
- **Document context** so we don't lose the reasoning behind past decisions
- **Enable async collaboration** across time zones and teams
- **Provide transparency** for anyone interested in how OpenShift Pipelines is built and maintained

Without this, knowledge lives only in people's heads or scattered across Slack threads and meeting notes.

## What belongs here?

This repository documents **product-level decisions** for OpenShift Pipelines - the Red Hat distribution of Tekton.

For Tekton feature proposals and API changes, see [Tekton Enhancement Proposals (TEPs)](https://github.com/tektoncd/community/tree/main/teps).

### In scope for this repo

- OpenShift integration (SCC, Routes, OAuth, Console plugin, OperatorHub)
- Support policy and lifecycle decisions
- Component inclusion/exclusion decisions
- Downstream-only patches and their rationale
- Release process and versioning
- Internal tooling and CI/CD decisions
- Security and compliance requirements (FIPS, CVE response)
- Konflux integration specifics
- Quality Engineering (QE) strategy and test architecture

### Out of scope (file a TEP or bring it up in upstream working groups)

- Tekton API changes (Pipeline, Task, TaskRun, etc.)
- Tekton Operator behavior (TektonConfig, addon management)
- New Tekton features
- Tekton Results, Chains, Hub, Dashboard features
- Anything that benefits the broader Tekton community

If you have an idea that belongs upstream, file a [TEP](https://github.com/tektoncd/community/tree/main/teps) or bring it up during the [Tekton working group meetings](https://github.com/tektoncd/community/blob/main/working-groups.md).

See [docs/scope.md](docs/scope.md) for detailed guidance.

## Structure

```
ADR/           # Architecture Decision Records
docs/          # Additional documentation
```

## Quick links

- [ADR Template](ADR/0000-adr-template.md)
- [Contributing Guide](CONTRIBUTING.md)
- [Scope Definition](docs/scope.md)
- [ADR Skill](skills/adr/SKILL.md)

## Authoring ADRs with AI Agents

This repository includes an interactive **ADR Assistant Skill** (`skills/adr/`) modeled after structured rubber-duck grilling to help stress-test architecture decisions, explore trade-offs, and draft ADRs.

AI coding agents (Claude Code, Pi, Cursor, Copilot, Codex, etc.) will read [`AGENTS.md`](AGENTS.md) when working in this repository and automatically know how to load and use the ADR skill.

You can also invoke it directly:
- **Claude Code**: Run `/adr <topic>` (or start with `--plugin-dir .`)
- **Pi**: Automatic via `AGENTS.md` or invoke `/adr <topic>`
- **Cursor / Copilot / Other Agents**: Automatic via `AGENTS.md` or simply ask:
  > *"Help me draft an ADR for `<topic>`."*

## Related

- [Tekton Enhancement Proposals](https://github.com/tektoncd/community/tree/main/teps)
- [konflux-ci/architecture](https://github.com/konflux-ci/architecture) (inspiration for this repo)
