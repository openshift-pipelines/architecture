---

## id: ADR-0004
title: Jenkins2Tekton Converter as an MTA Tackle2 Addon
date: 2026-10-11
status: proposed
authors:
  - "@ngelman1"
reviewers: []
tags:
  - migration
  - jenkins
  - tekton
  - mta

---

# ADR-0004: Jenkins2Tekton Converter as an MTA Tackle2 Addon

## Context

`Required`

Migration Toolkit for Applications (MTA) provides a comprehensive migration experience for application code: static analysis, replatforming, and AI-assisted refactoring. It does not address the migration of CI/CD pipelines themselves. A developer migrating an application must separately, manually convert their Jenkins pipelines to Tekton, a process that requires deep expertise in *both* platforms.

The [Jenkins2Tekton converter](https://github.com/ngelman1/Jenkins2Tekton_converter) is an existing AI-powered tool that automates this conversion using RAG with a vector store of Tekton documentation and Jenkins shared library context, driven by LLM. It produces valid Tekton v1 Pipelines, Tasks, and PipelineRuns from Jenkinsfiles and their shared library dependencies.

### Constraints

- Accurate conversion requires understanding the full Jenkins context: shared libraries (`vars/`, `src/`), multi-file job configurations, Groovy patterns, and plugin-specific steps.
- Accurate conversion also requires understanding the Tekton context: available Tasks from catalogs, pipeline patterns, workspace conventions, and best practices.
- The tool uses an LLM for conversion and a RAG pipeline for context — this requires network access and an LLM backend at runtime.
- MTA's Developer Lightspeed provides an existing LLM integration that can replace the tool's current Gemini dependency.

### Business Drivers

- **Product strategy**: Red Hat wants a single migration experience — developers using MTA should migrate both application code and CI/CD pipelines in one workflow.
- **Tekton adoption**: A high-quality conversion tool lowers the barrier to adopting OpenShift Pipelines by showing developers concrete output from their actual Jenkins jobs.
- **Reducing toil**: Manual conversion requires deep expertise in both Jenkins and Tekton. The tool eliminates this dual-expertise requirement.

## Decision

`Required`

Integrate the Jenkins2Tekton converter into MTA as a **Tackle2 Addon** — a container image registered with the MTA Hub that runs as a managed task, with full access to Hub-managed credentials, application inventory, and the MTA web UI for progress tracking and output viewing.

- Chosen option: `Tackle2 Addon` over `Crane Transform Plugin`, `Asset Generator Extension`, and `Standalone with Integration Hooks`.

### Architectural Overview

`Required`

```
+----------------------------------------------------+
|                   MTA Hub (Tackle2)                 |
|                                                     |
|  Application Inventory    Identity Store            |
|  Task Manager             REST API                  |
|  Web UI                                             |
|                                                     |
|  User triggers Jenkins2Tekton task for an app       |
|  Hub launches addon pod with credentials            |
+-------------------------+---------------------------+
                          |
                          | creates pod, injects secrets
                          v
+----------------------------------------------------+
|           Jenkins2Tekton Addon (Pod)                |
|                                                     |
|  1. Receive app metadata + credentials from Hub     |
|  2. Clone Git repo  OR  connect to Jenkins API      |
|  3. Ingest Jenkins shared library into RAG          |
|  4. Query Tekton docs RAG for target context        |
|  5. Build LLM prompt with full context              |
|  6. Call Developer Lightspeed for conversion         |
|  7. Validate YAML + LLM retry loop                 |
|  8. Report results back to Hub                      |
+-------------------------+---------------------------+
                          |
                          v
              Hub stores artifacts:
              - 00-pipeline-*.yaml
              - 00-task-*.yaml / 01-task-*.yaml
              - 00-pipelinerun-*.yaml
              - Conversion report (warnings, notes)
```

The addon runs in **Task mode**: the Hub creates a pod, injects a secret with the Hub URL and auth token, and the addon uses the `tackle2-addon` SDK to read application metadata and report results. The MTA web UI displays task progress, status, and output artifacts.

#### Input Sources

The addon supports two input sources:

1. **Git repository** — The Hub clones the application's Git repository (already part of the MTA application inventory). The addon reads Jenkinsfiles and shared library directories (`vars/`, `src/`, `resources/`) from the cloned source.
2. **Jenkins API** — The addon connects to a live Jenkins instance using Hub-managed credentials (Identity). It reads job configurations, shared library sources, installed plugins, and pipeline definitions directly from the Jenkins API.

### Implementation Details

`Required`

- **Addon container image**: Package the converter as a container image with the Python RAG pipeline, pre-built Tekton docs ChromaDB index, and a Go entry point using the `tackle2-addon` SDK.
- **Addon CR registration**: Define the Addon Custom Resource to register with the Hub, specifying task mode, resource requirements (CPU/memory for RAG and LLM calls), and PVC mounts for working directories.
- **Hub Identity type**: Define a Jenkins identity type in the Hub to store Jenkins instance URLs and credentials (username/token or API key).
- **Lightspeed integration**: Replace the Gemini SDK with the standard OpenAI Python client (`openai.OpenAI`) pointing at the in-cluster LLM proxy URL. Authentication uses the Hub JWT already provided by the addon SDK — no new credentials needed.
- **Micro-frontend (optional)**: Build a custom UI panel for the MTA web interface to display conversion-specific output (generated YAML with syntax highlighting, validation results, conversion notes).
- **Affected components**: MTA Hub (addon registration), Jenkins2Tekton converter (refactoring), MTA UI (optional micro-frontend).

## Other Approaches

`Required`

### Crane Transform Plugin

A crane plugin is a standalone binary that processes one exported Kubernetes resource at a time, offline, with no external access. It cannot work here because the converter needs to clone repos, access Jenkins instances, query RAG indexes, and call an LLM — none of which are possible in the offline, single-resource crane model.

### Asset Generator Extension (MTA 8.3 Generators)

MTA 8.3 generators are Helm-based templates that produce deterministic deployment artifacts. They are a poor fit because the Jenkins2Tekton conversion is AI-driven (LLM + RAG), requires long-running multi-step processing, and needs Jenkins-specific input sources that the generator framework does not support.

### Standalone Tool with MTA Integration Hooks

Keep the converter as its own project with MTA-compatible inputs/outputs. This was rejected because it provides no MTA UI integration (progress, status, output viewing), no Hub credential management, and leaves the tool undiscoverable to MTA users — missing the strategic goal of a unified migration experience.

## Testability

`Required`

*Open question — testing strategy to be determined. Options under consideration:*

The LLM-driven nature of the conversion introduces non-determinism. The testing strategy must account for output variability while still catching regressions in conversion quality.

## Operational Readiness

`Optional`

- **Resource requirements**: The addon requires more memory than typical static-analysis addons due to the ChromaDB vector store loaded in-memory. Resource requests/limits must be tuned.

## Scope

`Required`

- **In-Scope:**
  - Converting Jenkins Declarative and Scripted pipelines to Tekton v1 Pipelines, Tasks, and PipelineRuns
  - Resolving Jenkins shared library calls (`vars/`, `src/`) with full source context
  - Supporting input from Git repositories and live Jenkins instances
  - Referencing Tekton Hub catalog Tasks in generated output
  - LLM-driven conversion with RAG context and validation retry loop
  - Integration with MTA Hub (task lifecycle, credentials, artifacts, UI status)
  - Integration with Developer Lightspeed as the LLM backend
- **Out-of-Scope:**
  - Triggers and EventListeners — converting Jenkins webhooks/SCM triggers to Tekton Triggers
  - Credentials migration — moving Jenkins credentials into Kubernetes Secrets
  - Multibranch pipeline support — converting Jenkins multibranch pipelines to a Tekton equivalent
  - Bidirectional conversion — converting Tekton back to Jenkins
  - Replacing the standalone converter — it continues to work independently

## Acceptance Criteria

`Required`

The decision is considered accepted when:

- [ ] Problem, scope, and non-goals are clear
- [ ] Alternatives were considered and compared
- [ ] Chosen option's trade-offs (not just benefits) are documented
- [ ] Security model for Jenkins credentials is defined
- [ ] Testing strategy is defined (accounting for LLM non-determinism)
- [ ] Required reviewers approved
- [ ] Open concerns resolved or tracked
- [ ] Decision is feasible to implement within the Tackle2 addon framework

## Considered Options

`Required`


| Option                            | Pros                                                                                                                                                                                                                                                       | Cons                                                                                                                                                                                                                                                  |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Tackle2 Addon** (chosen)        | Native Hub UI integration (progress, status, output). Hub-managed credentials for Jenkins access. Lightspeed LLM available in-cluster. RAG bundled in container image. Established addon pattern (jkube, windup). Supports both Git and Jenkins API input. | Medium-high development effort. Requires Go entry point or shim around Python code. Must conform to addon SDK. Heavier resource footprint than typical addons (ChromaDB in-memory).                                                                   |
| Crane Transform Plugin            | Chains natively with the BuildConfig-to-Shipwright plugin (picks up skipped JenkinsPipeline BuildConfigs). Ships with `mta-ops` binary. Offline, deterministic. Lightweight.                                                                               | Only sees one Kubernetes resource at a time — cannot access shared libraries, Git repos, or Jenkins instances. No LLM/RAG (must work offline). No UI beyond CLI output. Fundamentally incompatible with the tool's context-aware, AI-driven approach. |
| Asset Generator Extension         | Integrates with MTA 8.3's existing Pipelines Generator framework.                                                                                                                                                                                          | Generators are Helm-template-based and deterministic — poor fit for AI-driven conversion. No progress tracking for long-running LLM tasks. No native Jenkins input support. Fights the generator architecture.                                        |
| Standalone with Integration Hooks | Lowest development effort. Full control over architecture.                                                                                                                                                                                                 | No MTA UI integration (no progress, status, or output viewing). Users must run the tool separately and manually import results. Misses the strategic goal of a unified migration experience. No Hub credential management.                            |


## Decision Drivers

`Required`

Key factors influencing this decision:

- **Product strategy alignment**: The tool must be part of the MTA migration experience, not a separate workflow.
- **UI integration**: Developers must see conversion progress, status, and output in the MTA web UI.
- **LLM/RAG requirement**: The tool's core value is AI-driven, context-aware conversion — any integration path that prevents LLM and RAG usage is disqualified.

## Consequences

`Required`

- Positive outcomes:
  - Developers get Jenkins-to-Tekton conversion inside their existing MTA migration workflow, and the tool becomes discoverable to all MTA users.
  - No API key management — the addon uses the Hub JWT and LLM proxy, credentials are platform-managed.
- Risks:
  - **Development effort**: Wrapping a Python RAG pipeline in a Go addon entry point adds complexity. The `tackle2-addon` SDK is Go-based; the converter is Python.
  - **LLM proxy dependency**: If the LLM proxy pod is unavailable or the administrator hasn't configured a provider, the addon cannot function. There is no deterministic fallback. The addon should surface a clear error when the proxy is unreachable.
  - **Resource footprint**: ChromaDB in-memory plus LLM API calls make this addon heavier than typical static-analysis addons.
- Technical debt or follow-up ADRs required:
  - Testing strategy definition
  - Micro-frontend design for rich output display in the MTA UI

## Dependencies

`Required`

- **MTA 8.x** with Tackle2 Hub addon support
- **Developer Lightspeed** configured and available in the MTA instance
- **Tackle2-addon SDK** (Go) for Hub integration

## Implementation Plan

`Required`

### Phase 1: Core Addon with Git Input

Build the minimal viable addon that handles the most common use case — converting Jenkinsfiles from a Git repository.

- Write the Go entry point (`main.go`) using the `tackle2-addon` SDK that shells out to the existing Python converter.
- Replace the Gemini SDK with the OpenAI Python client pointing at the in-cluster LLM proxy.
- Package the converter, pre-built ChromaDB index, and Go binary into a single container image.
- Register the Addon CR with the Hub and validate the end-to-end flow: user selects an application → triggers the addon → Hub launches the pod → addon clones the repo, runs conversion, reports results.
- **Deliverable:** A working addon that converts Jenkinsfiles from Git repositories, with task progress and output artifacts visible in the MTA UI.

### Phase 2: Jenkins API Input and Micro-Frontend

Extend the addon to support live Jenkins instances and improve the output experience.

- Add a Jenkins Identity type in the Hub so administrators can store Jenkins instance credentials.
- Implement Jenkins API client in the addon to fetch job configurations, shared library sources, and installed plugins directly from a running Jenkins instance.
- Build an optional micro-frontend panel for the MTA UI to display generated Tekton YAML with syntax highlighting, validation results, and conversion notes.
- **Deliverable:** Full addon supporting both Git and Jenkins API input sources, with a rich output viewing experience in the MTA UI.

## References

`Required`

- [Jenkins2Tekton converter repository](https://github.com/ngelman1/Jenkins2Tekton_converter)
- [MTA plugin primer — Writing an MTA plugin](https://psrvere.github.io/mta-plugin-primer/) — crane plugin model and design decisions for the BuildConfig-to-Shipwright plugin
- [Tackle2 Addon architecture (Konveyor)](https://hackmd.io/@konveyor/Hk2qTebWj) — Addon CR specification and Hub integration
- [tackle2-addon-jkube](https://github.com/konveyor/tackle2-addon-jkube) — reference addon implementation
- [MTA 8.3 release announcement](https://developers.redhat.com/articles/2026/10/08/migration-toolkit-applications-8-3-agentic-code-modernization) — OpenShift Pipelines Generator, agentic migration factory
- [MTA rules and providers](https://docs.redhat.com/en/documentation/migration_toolkit_for_applications/8.1/html/configuring_and_using_rules_for_an_mta_analysis/rules-prov-cond_rules-development-guide) — MTA analyzer architecture
- [konveyor/tackle2-hub](https://github.com/konveyor/tackle2-hub) — Hub REST API and task management
- [ADR-0001: Adopt ADRs](./0001-adopt-adrs.md)

## Change Log

`Required`


| Date       | Author    | Change Summary                  |
| ---------- | --------- | ------------------------------- |
| 2026-10-11 | @ngelman1 | Created initial ADR as proposed |


