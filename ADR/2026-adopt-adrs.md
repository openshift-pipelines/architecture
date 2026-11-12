# Adopt Architecture Decision Records

Date: 2026-02-09

## Status

Proposed

## Context

OpenShift Pipelines is a distribution of Tekton for OpenShift. We make many product-level decisions that are distinct from upstream Tekton development:

- Which OpenShift versions to support
- How to integrate with OpenShift-specific features (SCC, Routes, OAuth)
- Release and support policies
- Downstream patches and their rationale

These decisions are currently made informally and lack documentation. This makes it hard for new team members to understand why things are the way they are, and difficult to revisit past decisions.

Meanwhile, upstream Tekton has a well-established [TEP process](https://github.com/tektoncd/community/tree/main/teps) for feature proposals. We should not duplicate this - our process should complement it.

## Decision

We will use Architecture Decision Records (ADRs) to document product-level decisions for OpenShift Pipelines.

ADRs will be stored in the `openshift-pipelines/architecture` repository.

The scope is explicitly limited to product decisions. Tekton feature proposals belong upstream as TEPs.

## Consequences

### Benefits

- Decisions are documented and searchable
- New team members can understand historical context
- Clear separation between product decisions (ADRs) and feature proposals (TEPs)
- Lightweight process that doesn't slow down development

### Drawbacks

- Another repository to maintain
- Requires discipline to document decisions
- Some decisions may fall in gray areas between ADR and TEP

### Follow-up actions

- Archive or deprecate the stale `openshift-pipelines/enhancements` repository
- Announce the new process to the team
- Document first few ADRs to establish patterns
