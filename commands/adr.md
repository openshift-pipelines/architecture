# /adr - Architecture Decision Record Assistant

Use this command to create, stress-test, or update an Architecture Decision Record (ADR).

## Usage

```
/adr [topic or description]
```

## Behavior

1. Evaluates whether the proposed decision belongs in OpenShift Pipelines (downstream ADR) or Tekton Community (upstream TEP).
2. Interactively interviews you (one question at a time) to explore requirements, scope, options, trade-offs, dependencies, and quality considerations.
3. Automatically determines the next sequential ADR number.
4. Generates a fully formatted ADR file in `ADR/00XX-<topic>.md` adhering to `ADR/0000-adr-template.md`.
