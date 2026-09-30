---
name: adr
description: Interactively grill, refine, and author Architecture Decision Records (ADRs) for OpenShift Pipelines using the standard ADR template. USE WHEN user wants to create an ADR, propose an architectural change, evaluate design options, or stress-test an architectural decision.
---

# Architecture Decision Record (ADR) Co-Author & Grilling Skill

Interactively guide and stress-test the creation of OpenShift Pipelines Architecture Decision Records (ADRs). Modeled after rubber-duck grilling: surface hidden assumptions, force trade-offs into the open, verify upstream vs. downstream scope, and produce a high-quality ADR following the repository template.

## Workflow Phases

```
1. Scope & Upstream Check  →  2. Grilling & Decision Tree  →  3. Template Drafting  →  4. Verification
```

> **Note on Iteration**: ADRs do not need to be 100% complete or `Accepted` before merging into `main`. Merging early with `status: proposed` is encouraged to gain team visibility, solicit async reviews, and iteratively fill out implementation details, testability, or operational readiness in subsequent PRs.

---

## Phase 1: Scope & Upstream Check

Before drafting, ensure the decision belongs in this repository:
- **Upstream (TEP)**: Tekton API changes, core controller behaviors, new features benefiting general Tekton community → *Direct user to upstream Tekton Enhancement Proposals (TEPs)*.
- **Downstream (OSP ADR)**: OpenShift integrations (OAuth, SCC, Console plugin, OperatorHub), downstream packaging/distribution, Konflux CI/CD, support policies, downstream-only patches.

Check for confidentiality: ensure no customer names or confidential internal identifiers are used (use generic descriptions: "large multi-tenant cluster", "disconnected enterprise environment").

---

## Phase 2: Grilling (Structured Exploration)

Read `ADR/0000-adr-template.md` first to identify all required and optional sections. Use the template structure to dynamically drive the interactive interview and decision tree:

### Rules for Grilling
- **One question at a time.** Never overwhelm the author with a giant questionnaire.
- **Provide your recommended answer/default** with brief reasoning so the author can simply confirm with "yes" or adjust.
- **Explore available repo context first** before asking questions that existing docs or code already answer.
- **Pursue dependencies depth-first**: Walk down each branch of the decision tree (exploring constraints, trade-offs, and operational impacts) before moving to the next section.
- **Template-Driven Inquiry**: Dynamically probe every section defined in `ADR/0000-adr-template.md`:
  - *Context & Drivers*: Why is this decision needed now? What technical or organizational constraints exist?
  - *Decision & Architecture*: What is the chosen solution? High-level design and affected components?
  - *Scope*: What is explicitly in-scope vs. deliberately out-of-scope?
  - *Considered Alternatives*: What other options were considered (including "Do Nothing") and why were they rejected?
  - *Consequences & Trade-offs*: What are the positive outcomes, risks, trade-offs, and technical debt?
  - *Non-Functional & Operational Requirements*: Dependencies, compatibility, security/RBAC, observability, testing, rollback (as defined in the template).

---

## Phase 3: Incremental Drafting

Once alignment is reached, generate the ADR adhering to the repository's template:

- **Location**: `ADR/00XX-<short-slug>.md` (next sequential number after checking existing ADRs in `ADR/`).
- **Template Compliance**: Read `ADR/0000-adr-template.md` directly as the single source of truth for document structure. Fill in all required sections as defined in the template, and include any applicable optional sections based on the grilling phase.
- **Metadata**: Populate all metadata fields specified in the template.

---

## Phase 4: Review Checklist

Verify the completed ADR against this checklist before finalizing:
- [ ] Belongs in OSP ADRs (not upstream TEPs)
- [ ] No customer names or confidential data
- [ ] All required sections from `ADR/0000-adr-template.md` are completed
- [ ] Scope clearly separates goals from non-goals
- [ ] Alternatives (including "Do Nothing") evaluated
- [ ] Both benefits and drawbacks honestly documented in Consequences
