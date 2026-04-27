# 0002. "Hack" Release Automation Design

Date: 2026-04-24

## Status

Proposed

<!-- Status lifecycle:
- Proposed: Under discussion, not yet accepted
- Accepted: Approved, ready for implementation
- Implemented: Done and reflected in the product
- Superseded: Replaced by a newer ADR (link to it)
- Rejected: Not accepted (stays in repo for reference)
-->

## Context

Creating a new Release in the Hack repository can be buggy and tedious to verify.
Buggy because the current source of truth, the ["Release Files"][release-files], are automatically generated based on the latest upstream component's versions which may be ahead of the Operator's versions.
Tedious to verify because a given component's version is set in [the Release Files][release-files] and (in most but not all cases) in the upstream Operator's [components file][operator-components]; automation may cause the former to drift so every component has to be verified.
The goal is balancing automation to reduce toil and fundamentally required manual maintenance, and at the same time creating a single source of truth for the component versions included in a given release.

### Constraints
- There must be a source of truth for the component versions included in a Release.
- Five categories of downstream components, in terms of versioning:
    - The Operator: version must be explicitly set downstream.
    - Components included in Operator: versions must be aligned with the version included in the Operator (e.g. Tekton Results).
    - Components not included in the Operator with their own versions: versions must be explicitly set downstream (e.g. Tkn CLI).
    - Components not included in the Operator with versions that align with the OSP version: inclusion in the release must be explicit, version can be inferred (e.g. OPC).
    - Components not included in the Operator which are versioned using branches instead of releases: branch must be specified downstream.
- The components included in a release must be the same as those included in the included Operator version.
- Without adding unnecessary toil, the versions of components which are not included by the upstream operator must be defined manually.
- The definitions must be unambiguous about which versions are included in a given release.
- The release automation design must be accepted by the team since it will be engaged with by all Release Captains.

### Current State
Currently, the Release Captain runs the GitHub Action [Automated Release Actions][release-actions-action] selecting the action "new release".
This runs automation which creates a new Release File which enumerates every component and their latest upstream version.
The Release Captain must then check that those components which are included in the upstream Operator's components file are aligned with the versions listed there; a component's latest release may not have yet been included in an Operator release.
As the release is iterated, sometimes changes are made to which versions are used and the Release Captain must ensure the Release File is updated in these instances, especially with Manual Approval Gate and the `tkn` CLI.

Example of a Release File, the `branches` key is relevant to this design:
```yaml
version: "1.20"
image-suffix: -rhel9
patch-version: 1.20.4
code-freeze: true
branches:
  git-init:
    upstream: release-v1.2.x
  manual-approval-gate:
    upstream: release-v0.7.0
  opc:
    upstream: release-v1.20.x
  pipelines-as-code:
    upstream: release-v0.37.x
  tekton-assist:
    upstream: release-v0.1.x
  tekton-caches:
    upstream: release-v0.3.x
  tektoncd-chains:
    upstream: release-v0.25.x
  tektoncd-cli:
    upstream: release-v0.42.1
  tektoncd-hub:
    upstream: release-v1.22.10
  tektoncd-pipeline:
    upstream: release-v1.3.x
  tektoncd-pruner:
    upstream: release-v0.2.x
  tektoncd-results:
    upstream: release-v0.16.x
  tektoncd-triggers:
    upstream: release-v0.33.x
  operator:
    upstream: release-v0.77.x
```

#### Pain Points

The Release Files themselves are not the issue.
The way they fit into our workflow results in pain points:

- The Release Files are the "source of truth" for the rest of the automation, but since they are automatically created from upstream releases they can drift from the Operator
- Manually auditing the Release Files is tedious since they enumerate every component, and not all components use the same release-branching strategy
- The Release Files list the upstream **branch** to be used making it less obvious to developers and LLMs that it can be used as the source of truth
    - A few components use patch versions in their branch, but most use just the minor version.
- While the Release Files necessarily list all component versions, only three pieces of information are actually required in theory
    - The Operator upstream version
    - The versions for components not included by the Operator
    - The versions of components which are different than the version in the operator (rarely relevant outside of nightly builds)

### Proposal

Create a single Releases (plural) file which acts as the source of truth of all releases and contains a minimally complete definition of every component included per release.
All components included in the Operator's `components.yaml` have their versions inferred from the Operator.
This file will be _manually maintained_ to avoid automation causing drift, but will be terse enough to avoid unnecessary toil.

The Releases File contains a list of Releases, and each release lists necessary components and their respective versions:
- The Operator version is listed.
- Components which are included in the Operator's `components.yaml` don't need a specified version, it's inferred from the Operator. Specifying these components and their versions overrides the version coming from the Operator.
- Components which are versioned using a branch (e.g. git-init) specify `branch: <branch name>`.
- Components which are versioned using the OSP version (e.g. OPC-CLI) do not need to specify a version, the version is inferred from the OSP version.
- Components which are not included in the Operator's `components.yaml` but need a specific version specify their version with `version: <minor-version>` (e.g. `version: 1.2`).

```yaml
releases:
  - version: "1.22"
    components:
      - name: operator
        version: "0.79"
      - name: git-init
        branch: "main"
      - name: tekton-cli
        version: "0.44"
      - name: tekton-assist
        branch: "main"
      - name: caches
        version: "0.3"
      - name: opc
        # version inferred
      - name: console-plugin
        # version inferred
  - version: "1.23"
    components:
      - ...
```

Alternatively, a terser schema would also work, as demonstrated below.
In such a case, an inferred component version would be noted with a sentinel like "-" (to differentiate from zero values), and automation would be responsible for differentiating when a version is branch like "main" versus a minor version like "1.2".

```yaml
releases:
  - version: "1.22"
    components:
      operator: "0.79"
      git-init: "main"
      tekton-cli: "0.44"
      tekton-assist: "main"
      caches: "0.3"
      opc: "-"
      console-plugin: "-"
  - version: "1.23"
    components:
      - ...
```

## Decision

### 1. Couple Upstream Component Versions to Downstream Releases

The primary decision necessary is to use the upstream Operator's `components.yaml` as the source of truth for all applicable component versions.
Component version overrides can still be made if necessary, but are uncommon.

### 2. Adopting "Releases" File

Adopt the `releases.yaml` config file which defines the component versions in the release, sans operator-specified versions.
Number 1 may be adopted alone by changing the automation for generating the existing Release Files to pull from the latest Operator's `components.yaml`.

## Consequences

When a Release Captain starts the process of creating a new release, they'll do so by adding a new release in the Releases File, specifying the Operator version and any component versions for components not included in the Operator.
Automation will consume this file and merge it with the Operator's `components.yaml` to produce a Release File of the current format.
This file can be consumed by, and eventually modified by, an LLM and/or dashboard.

### Benefits

1. Component versions cannot drift from the upstream Operator's version without an explicit override (which can result in a CI warning).
1. The manual maintenance of a release is minimal but complete.
1. A (machine-parsable) source of truth exists that defines all versions of our components for any given release, which can inform other systems, dashboards, etc.
1. This change would be completely isolated in the hack repository and does not require changes to any other downstream repositories.

### Drawbacks

- The Releases File doesn't specify the upstream components' versions explicitly. This has benefits but it does mean the file alone doesn't list every component version, you have to merge or see the merged `components.yaml` file.
- Another file means added complexity to the automation.
- The file encodes state, not semantics. So when a downstream component is downstream only and then gets included in the Operator it disappears off the list which can cause confusion if the file is referenced without the corresponding `components.yaml`.
- A YAML or JSON file is just a file, but here it's being used in place of a database, which has inherent drawbacks (e.g. implicit validation, relationships, etc).

### Follow-up actions

- Enumerate the existing maintained releases into the new Releases File.
- Extend the automation which generates Release Files to do so by merging the Releases File and the Operator's component versions.
- Update the Release Captain documentation with the new workflow.

[release-actions-action]: https://github.com/openshift-pipelines/hack/actions/workflows/release-manager.yaml
[release-files]: https://github.com/openshift-pipelines/hack/tree/main/config/downstream/releases
[operator-components]: https://github.com/tektoncd/operator/blob/main/components.yaml
