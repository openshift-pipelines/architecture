---
id: ADR-0005
title: Agentic Diagnosis for Failed PipelineRuns
date: 2026-09-02
status: proposed
authors:
  - "@waveywaves"
reviewers:
  - "@jkhelil"
tags:
  - pipelines
  - agentic-diagnosis
---

# ADR-0005: Agentic Diagnosis for Failed PipelineRuns

## Context

OpenShift Pipelines users working in the console need to understand failed `PipelineRun`s. Existing Lightspeed chat with Tekton MCP tools is the baseline: a user can already ask for an investigation. Another entry point is useful only if it makes that journey easier, for example by preserving the selected workload context and exposing an asynchronous analysis that the user can return to.

The [Lightspeed Agentic Operator](https://github.com/openshift/lightspeed-agentic-operator) manages agent workflows, sandbox lifecycle, approvals, and results. An `AgenticRun` is its namespaced custom resource describing a request and the agents used for workflow steps. Analysis is required; execution and verification are optional. An analysis-only run omits those latter steps. This is the existing Lightspeed API, not a new Pipelines-owned agent abstraction.

Agentic diagnosis covers both console-initiated and automatically initiated analysis. The design discussion favored first refining the console experience with UI owners. Asynchronous analysis after a user action and automatic discovery of failures are different decisions. A background watcher is not required for the former. AgentTask is not required for either direct submission to Lightspeed. Existing invocation prototypes are evidence to reuse, not proof of a supported console integration.

Pipelines-as-Code (PaC) and Paco serve the developer's repository and pull-request workflow, including provider reporting. This proposal targets console users investigating workloads on the platform. Requiring PaC/Paco to manage AgenticRun submission, completion, and reports would add integration work without an agreed consumer benefit. Reuse is optional, not a prerequisite.

## Decision

Use the existing Lightspeed AgenticRun API and Tekton MCP capabilities as the analysis-only contract for agentic diagnosis, whether initiated from the console or by a future automatic producer. First agree the console invocation, status, result, and presentation contract with UI and Lightspeed owners and demonstrate its value over existing chat. Keep automatic failure discovery as a separate follow-up decision rather than commit to a new controller now.

### Architectural Overview

The candidate flow is:

1. An authorized user selects a failed PipelineRun in the Pipelines console and requests diagnosis.
2. The agreed console integration submits or reuses an equivalent analysis-only AgenticRun, bound to the source cluster, namespace, name, and UID.
3. The Agentic Operator manages the analysis independently of the browser session. Its sandbox uses the configured model and authorized Tekton MCP tools to retrieve bounded evidence.
4. The console observes workflow status, retrieves the authorized analysis result, and presents it or navigates to the agreed shared detail view. Reopening the page does not itself start another analysis.

No new controller or intermediary API is selected for this first journey. The owners must choose the smallest supported console-to-Lightspeed integration path; a new backend service is not assumed. The console handles user interaction, while Lightspeed owns the submitted workflow and result lifecycle. Neither performs workload remediation as part of this proposal.

A future automatic producer could submit the same diagnosis contract without a user click. Its failure selection, enrollment, durable deduplication, and admission/cost controls need a separate decision; asynchronous AgenticRun execution alone does not supply automatic discovery.

### Implementation Details

- Agree action placement and availability, the submitting identity, source binding, AgenticRun placement, and the supported API with UI and Lightspeed owners. Same-namespace placement is a candidate, not a verified deployment contract.
- Submit bounded intent and source identity, not copied logs or credentials. Reuse the Tekton MCP evidence path rather than build another pipeline, networking, or registry diagnosis engine.
- Use supported AgenticRun conditions for progress and result references to retrieve analysis. The current API references AnalysisResult resources through analysis-step status; do not assume the full report is inline in AgenticRun status. Agree how the console chooses and displays a result or a failed attempt.
- Define active-run reuse, duplicate-click and lost-create-response handling, explicit re-analysis, retention, and source/result deletion. Opening a page or losing a result must not silently request paid inference; a deliberate retry is a separate user action.
- Define the result/detail route, pending and denied states, failure and expiry presentation, and return-to-analysis behavior. Reuse or adapt the shared AgenticRun view if it serves the journey; do not create a generic chat UI or hard-code an unagreed route.
- Validate configured provider, model, MCP, credentials, limits, and supported component versions. Existing Lightspeed installation is a prerequisite to check, not something every Pipelines customer can be assumed to have. Reusing its configuration does not imply inference is free or requires no credentials.

## Security & Compliance

- **Prevent privilege escalation:** use short-lived analysis identities and trusted source binding. Namespace-scoped read access is not source-scoped authorization; independently enforce the PipelineRun UID and related-resource ownership at every evidence access path. A prompt or model-supplied resource name is not an authorization boundary.
- **Constrain tools and credentials:** expose only the approved read-only analysis capabilities. Deny Secret access, workload mutation, Pod execution, unrelated workloads, and cross-namespace access. Toolset enablement alone does not impose read-only behavior on a shared MCP server. Credential injection belongs to the trusted runtime, never the request, prompt, status, or report.
- **Constrain data and egress:** treat logs and workload content as untrusted input. Limit evidence and approved destinations, enforce network policy, and document provider data handling. Byte limits and redaction are not guarantees that sensitive data cannot leave the cluster.
- **Verify transport:** use the supported operator/sandbox TLS contract, preserving certificate-chain and hostname verification and failing closed on invalid required trust. The current sandbox design uses a shared system-plus-custom CA bundle, not isolated per-server trust. This ADR does not reintroduce a separate `caFile` implementation; acceptance requires a security decision on that shared trust boundary.
- **Protect results:** reports can contain sensitive evidence. Authorize result, status, event, console, and report access at least as strictly as source log access. Authorize submission and result retrieval independently; showing or hiding a button is not an authorization boundary.

## Compatibility & Migration

The proposed action must not change PipelineRun outcomes or remove existing Lightspeed chat, AgentTask invocation, or PaC/Paco workflows. Enable it only for an explicitly configured, supported Lightspeed integration; current source APIs are not evidence of a released product combination.

Disabling the integration stops new submissions. Agree how to handle active analyses and retained results on disablement or upgrade. Missing dependencies make the action unavailable or produce a clear diagnosis-specific error without degrading otherwise healthy core Pipelines.

## Testability

Validate the journey with UI stakeholders against existing Lightspeed chat: selected-workload context, steps required to start diagnosis, asynchronous progress, and the ability to return to an existing result. Any reduction in repeated work or load is a hypothesis to measure, not an established benefit.

Unit and integration tests cover action availability, authorized creation/reuse, UID binding, duplicate clicks, lost responses, page closure/reopening, result-reference resolution, failure, denial, expiry, and source deletion. Released-stack tests must prove analysis-only behavior, source-scoped evidence access, Secret and unrelated-resource denial, protected report access, evidence/egress bounds, TLS verification, and unavailable-dependency behavior. Record exact source and image versions and distinguish source checks, prototypes, and real model-backed validation.

## Operational Readiness

Agree accountable owners for the console action, integration API, runtime compatibility, and support. Observe request failures, active analysis, completion, result availability, and usage without logging credentials or evidence. Keep diagnosis availability separate from core Pipelines health. The console must not imply analysis succeeded merely because the AgenticRun was created.

## Scope

- **In scope:** the shared agentic diagnosis contract for console or automatic entry points, starting with the console use case, its value over chat, analysis-only request/status/results, authorization, and UI/runtime ownership.
- **Out of scope:** committing a background failure watcher, automatic remediation, execution, a new diagnostic engine or generic agent API, mandatory AgentTask adoption, replacing PaC/Paco, or moving Git-provider checks/comments into AgenticRun. This ADR does not itself deliver the console implementation.

## Acceptance Criteria

- [ ] UI, Pipelines, and Lightspeed stakeholders agree the target console user and demonstrate a useful journey beyond existing chat.
- [ ] Submission, source identity, AgenticRun placement, asynchronous status, result references, and the display/detail-view contract are agreed with accountable owners.
- [ ] Duplicate submission, active-analysis reuse, explicit re-analysis, browser disconnect, expiry, failure, and unavailable-dependency behavior are defined.
- [ ] Source-bound evidence authorization, protected result visibility, credentials, egress, and the shared CA trust boundary have security agreement.
- [ ] Supported component versions, configuration prerequisites, disablement, observability, and support responsibilities are validated.
- [ ] Required reviewers approve the refined proposal; automatic triggering and optional PaC reuse are recorded as separate decisions, not implied delivery commitments.

## Considered Options

| Option | Advantages | Trade-offs and disposition |
| --- | --- | --- |
| Existing Lightspeed chat / no new entry point | Already offers diagnosis through MCP without another integration | Baseline and valid outcome if the proposed action adds insufficient value. |
| Console action using the existing AgenticRun API | Preserves workload context and exposes a separately managed asynchronous analysis | First candidate to refine; needs an agreed invocation/result contract and evidence that the journey is useful. |
| Automatic failure producer | Could prepare analysis before a user opens the console | Separate follow-up: needs a justified trigger policy, durable deduplication, admission/cost controls, and a support owner. No watcher is selected now. |
| AgentTask intermediary | Useful when an explicit Tekton workflow needs interchangeable agent backends | Adds an unnecessary layer to direct console submission; not required here. |
| PaC/Paco investigation | Serves repository workflows with their existing trigger and report-publishing model | Keep independent; optional Lightspeed reuse requires a consumer benefit that outweighs integration complexity. |
| Core Tekton or Agentic Operator discovers failures | Reuses an existing controller process | Adds optional diagnosis policy to core execution or product-specific discovery to the shared runtime; not needed for the console-first journey. |

## Decision Drivers

Improve the console investigator's journey without duplicating Lightspeed diagnosis; reuse existing MCP work and configured infrastructure; keep the integration small; preserve authorization and provider choice; establish a clear request/result contract before adding automatic discovery.

## Consequences

### Benefits

- A user could request diagnosis in the failed PipelineRun's context and return to the analysis later.
- Lightspeed supplies the diagnosis workflow and Tekton MCP supplies evidence; Pipelines need not build another investigation engine.
- The first investigation can proceed without a new watcher, a generic adapter, or a PaC migration.

### Drawbacks

- The console integration may duplicate chat unless its user value is demonstrated, and it creates a cross-team API and support dependency.
- User-requested inference still consumes model capacity and can send workload data to a configured provider.
- Shared CA trust is broader than the earlier per-server proposal. Namespace RBAC alone also does not deliver source isolation; both require explicit security treatment.
- Returning to existing analysis requires agreed result retention, access, and stale/deleted-source handling. Evidence may disappear through normal workload pruning.

### Follow-up actions

Work with UI owners on the invocation/status/result/display contract and compare the journey with existing chat. Refine asynchronous behavior in the proposed follow-up discussion. Revisit automatic discovery only with a concrete additional use case, trigger/cost/lifecycle policy, and owner. Consider PaC/Paco reuse only when a real consumer need justifies it.

## Dependencies

Requires console and Lightspeed owner agreement, compatible Agentic Operator/sandbox/MCP delivery, configured provider and credential wiring, approved analysis identity and source authorization, and protected result retrieval. Reuse the shared detail surface where suitable. Shared TLS configuration and toolset enablement are capabilities to validate, not proof that stronger isolation requirements are implemented.

## References

- [AgenticRun API and analysis-only workflow](https://github.com/openshift/lightspeed-agentic-operator/blob/main/api/v1alpha1/agenticrun_types.go)
- [AgenticRun status and AnalysisResult references](https://github.com/openshift/lightspeed-agentic-operator/blob/main/api/v1alpha1/agenticrun_status_types.go)
- [AgentTask/Lightspeed prototype evidence and gaps](https://github.com/openshift-pipelines/agenttask-adapter-lightspeed/issues/2)
- [Related PaC follow-up trigger proposal](https://github.com/openshift-pipelines/architecture/pull/14)
- [Earlier PaC investigation ADR, closed for a more scoped proposal](https://github.com/openshift-pipelines/architecture/pull/12)
- [ADR discussion and review](https://github.com/openshift-pipelines/architecture/pull/5)

## Change Log

| Date | Author | Change |
| --- | --- | --- |
| 2026-09-02 | @waveywaves | Initial proposal. |
| 2026-11-17 | @waveywaves | Address reviewer feedback on terminology, alternatives, security boundaries, lifecycle gaps, and current scope. Pending stakeholder review. |
| 2026-11-17 | @waveywaves | Refine after design discussion: console-first request/result contract, asynchronous analysis distinct from automatic triggering, no mandatory AgentTask or PaC/Paco integration. Still proposed. |
