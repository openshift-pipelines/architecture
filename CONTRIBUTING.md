# Contributing to OpenShift Pipelines Architecture

## Before you file an ADR

Ask yourself: **Does this belong upstream?**

If your proposal affects Tekton APIs, behavior, or features that other Tekton users would benefit from, file a [TEP](https://github.com/tektoncd/community/tree/main/teps) or bring it up during the [Tekton working group meetings](https://github.com/tektoncd/community/blob/main/working-groups.md).

This repo is for **OpenShift Pipelines product decisions** only. See [docs/scope.md](docs/scope.md) for details.

## When to file an ADR

- OpenShift-specific integration decisions
- Support policy or lifecycle changes
- Decisions about what components to include/exclude
- Downstream patch rationale
- Release or versioning process changes
- Security/compliance requirements

## Process

### 1. Create your ADR

You can draft your ADR manually or use your AI agent (which automatically reads [`AGENTS.md`](AGENTS.md) and loads the built-in skill `skills/adr/SKILL.md`):

**Option A: Using an AI Agent (Claude Code, Pi, Cursor, etc.)**
```bash
# In Claude Code or Pi:
/adr "My proposed decision"
# Or in Cursor/Copilot:
"Help me draft an ADR for <my proposed decision>"
```

**Option B: Manual copy**
```bash
cp ADR/0000-adr-template.md ADR/00XX-my-decision.md
```

Use the next available number. Fill in all sections.

### 2. Open a Pull Request

- Title: `ADR: <brief description>`
- Fill in the PR template
- Request review from maintainers

### 3. Discussion period

- ADRs are discussed in PR comments
- Significant ADRs may be announced in team meetings
- Address all feedback

### 4. Lifecycle, Review & Acceptance

- **Iterative Drafting**: ADRs do **not** need to be fully complete or `Accepted` to be merged. Authors are encouraged to open and merge early drafts with `status: proposed` once the context, scope, and initial shape are captured, and iterate collaboratively on design details, alternatives, and consequences in follow-up PRs.
- **Discussion**: ADRs are discussed in PR comments and team meetings.
- **Acceptance**: Once the decision reaches consensus and fulfills the acceptance criteria defined in the template, a PR transitions the ADR status from `proposed` to `accepted`. When implemented in the product, it moves to `implemented`.

## Style guidelines

- Keep it concise - ADRs should be readable in a few minutes
- Focus on **why**, not just what
- Be honest about trade-offs in Consequences
- Link to relevant TEPs, issues, or external docs

## Confidentiality

**Do not include customer names in ADRs.** This is a public repository.

If a decision is driven by customer feedback or requirements:
- Describe the requirement generically (e.g., "users need...", "a common request is...")
- Reference internal tickets by number if needed, without customer details
- Focus on the technical problem and solution, not who asked for it

## Questions?

Open an issue if you're unsure whether something belongs here or upstream.
