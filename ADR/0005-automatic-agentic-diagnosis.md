---
id: ADR-0005
title: Automatic Agentic Diagnosis for Failed PipelineRuns
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

# ADR-0005: Automatic Agentic Diagnosis for Failed PipelineRuns

## Context

OpenShift Pipelines users diagnose failed `PipelineRun`s by correlating Tekton status, events, Pod status, and logs. Automatic diagnosis could reduce that work for enrolled teams, but also consumes model capacity and can send workload data to a configured provider. Whether this justifies a separately supported producer remains a product and engineering decision.

The [Lightspeed Agentic Operator](https://github.com/openshift/lightspeed-agentic-operator) manages agent workflows, sandbox lifecycle, approvals, and results. An `AgenticRun` is its namespaced custom resource describing a request and the agents used for workflow steps. Analysis is required; execution and verification are optional. An analysis-only run omits those latter steps. The operator manages submitted runs; discovery of failed Tekton workloads belongs to a domain-specific producer.

Three invocation paths must remain distinct: explicit AgentTask invocation, a user action from the console, and automatic failure discovery. A successful invocation prototype does not establish automatic triggering or released-product readiness. Pipelines-as-Code (PaC) already has a separately tracked follow-up investigation design for repository workloads; this proposal must justify any additional producer rather than replace that work.

## Decision

Propose a standalone, Pipelines-owned controller for opt-in, analysis-only diagnosis of failed and timed-out PipelineRuns not already handled by an enrolled investigation flow. Prefer this over changing core Tekton controllers or using the console as a background watcher. Implementation is conditional on agreement about the use case, ownership, overlapping producers, and the acceptance gates below.

### Architectural Overview

The producer observes eligible terminal PipelineRuns and submits a source-linked AgenticRun in the source namespace. The Agentic Operator owns analysis execution and result lifecycle; the producer owns failure classification, enrollment, source identity, deduplication, and delivery status. Neither the model nor the producer performs workload remediation.

Cluster administrators authorize the feature and approved providers; namespace administrators enroll their namespaces within that policy. A source-level opt-out takes precedence. Namespace enrollment must not grant access to an unapproved provider or permit a namespace user to widen cluster policy.

The producer passes namespace, name, UID, and bounded intent, not copied logs or credentials. Authorized tools retrieve bounded evidence. The result consumer is the authorized investigator through an agreed AgenticRun result/detail surface; console navigation is a separate integration decision, not a prerequisite for a background producer.

### Implementation Details

- The earlier `TektonConfig.spec.agenticDiagnosis`, namespace label, annotation, and readiness-condition spellings are API proposals, not existing supported contracts. Define their schema, validation, ownership, and compatibility before implementation.
- Classify terminal failures and timeouts explicitly; exclude cancellations, graceful stops, diagnosis workloads, and sources claimed by another enrolled investigation path. Agree a bounded lookback so enablement or restart does not analyze an unlimited history.
- Use source cluster, namespace, and PipelineRun UID as the identity, not name alone. Persist admission and outcome state in Kubernetes so restart, create-response loss, result expiry, or manual result deletion cannot silently admit the same automatic diagnosis again. Deterministic names alone do not satisfy this requirement.
- Choose the durable record and its retention before implementation. Do not assume an AgenticRun survives its own TTL, and do not add source annotations while claiming the source is never mutated. Any bookkeeping-write permission needs an explicit review.
- Bound active work per namespace and cluster, evidence bytes, analysis duration, and admission rate. Admission must remain safe across concurrent reconciles and replicas; count-then-create is not sufficient. Coordinate producer backpressure with receiver-side intake governance rather than creating a second generic queue.
- Define retryable infrastructure failures separately from completed analysis attempts, with bounded retry and age limits. Preserve deferred work through restart without an unbounded in-memory backlog.

## Security & Compliance

- **Prevent privilege escalation:** use short-lived analysis identities and trusted source binding. Namespace-scoped read access is not source-scoped authorization; independently enforce the PipelineRun UID and related-resource ownership at every evidence access path. A prompt or model-supplied resource name is not an authorization boundary.
- **Constrain tools and credentials:** expose only the approved read-only analysis capabilities. Deny Secret access, workload mutation, Pod execution, unrelated workloads, and cross-namespace access. Toolset enablement alone does not impose read-only behavior on a shared MCP server. Credential injection belongs to the trusted runtime, never the request, prompt, status, or report.
- **Constrain data and egress:** treat logs and workload content as untrusted input. Limit evidence and approved destinations, enforce network policy, and document provider data handling. Byte limits and redaction are not guarantees that sensitive data cannot leave the cluster.
- **Verify transport:** use the supported operator/sandbox TLS contract, preserving certificate-chain and hostname verification and failing closed on invalid required trust. The current sandbox design uses a shared system-plus-custom CA bundle, not isolated per-server trust. This ADR does not reintroduce a separate `caFile` implementation; acceptance requires a security decision on that shared trust boundary.
- **Protect results:** reports can contain sensitive evidence. Authorize result, status, event, console, and report access at least as strictly as source log access. Admission, RBAC, tool authorization, and result access must agree; a namespace label is not permission to expose reports.

## Compatibility & Migration

The feature is disabled by default and must not change existing PipelineRun outcomes or manual invocation. The producer must use a tested, versioned Agentic Operator, sandbox, and MCP combination; current source APIs are not evidence of a supported product combination.

Disabling enrollment stops new admissions. Before enablement, define how to stop or drain already admitted analysis, revoke credentials, and preserve protected results. Missing dependencies disable diagnosis visibly without degrading otherwise healthy core Pipelines.

## Testability

Unit tests cover terminal-reason classification, enrollment precedence, lookback, source UID identity, and bounded retry. Integration tests cover concurrent admission, lost create responses, controller restart, deferred-work recovery, result expiry/deletion, source deletion, and overlapping producers.

Released-stack tests must prove analysis-only behavior, unrelated-resource and Secret denial, no alternative tool bypass, result-access restrictions, evidence/egress bounds, TLS verification and CA rotation, dependency failure, and disable/drain behavior. Record exact source and image versions and distinguish source checks, fixture runs, and real model-backed validation.

## Operational Readiness

Expose diagnosis readiness separately from core Pipelines readiness, with visible skipped, deferred, denied, and failed outcomes. Monitor backlog age, active analyses, admission denials, dependency errors, and model usage without logging credentials or evidence. Pipelines owns producer packaging, upgrades, support, and security response; Lightspeed owners retain the runtime contracts. Named accountable owners and a compatibility/support policy are acceptance requirements, not assumed commitments.

## Scope

- **In scope:** the decision whether to add automatic discovery, its producer boundary, enrollment, analysis-only authorization, lifecycle, and operational requirements.
- **Out of scope:** automatic remediation, execution, new upstream Tekton APIs, console implementation, replacing PaC/Paco, or selecting AgentTask as a mandatory intermediary. Optional interoperability is separate discovery.

## Acceptance Criteria

- [ ] Product and engineering agree the automatic use case, the gap beyond manual/PaC flows, and accountable producer/runtime owners.
- [ ] The source namespace, identity, analysis-only AgenticRun contract, and supported component versions are validated.
- [ ] Source-bound evidence authorization, protected result visibility, credentials, egress, and the shared CA trust boundary have security agreement.
- [ ] Durable deduplication storage, retention, atomic admission, bounded lookback/retry, producer coexistence, and disable/drain semantics are decided and testable.
- [ ] Configuration/status API, compatibility, packaging, observability, and support responsibilities are agreed.
- [ ] Delivery follows the current discovery decision; completed MCP work is not reopened or used as evidence that this producer exists.

## Considered Options

| Option | Advantages | Trade-offs and disposition |
| --- | --- | --- |
| Manual diagnosis only / no new producer | No new background component or automatic model cost | Valid baseline and defer/no-go outcome; does not start unattended analysis. |
| Reuse PaC follow-up investigation | Reuses repository opt-in, failure context, and report publishing | Prefer for its existing repository flow; does not by itself cover every non-PaC PipelineRun. Establish the uncovered use case first. |
| Standalone Pipelines producer | Owns Tekton-specific discovery without changing upstream execution semantics | Proposed for an agreed gap; adds a component, admission state, RBAC, packaging, and support cost. |
| Core Tekton controller integration | Sees terminal state directly | Couples core execution to optional inference policy and downstream runtime dependencies; not preferred. |
| Console background discovery | Reuses a visible user surface | Browser sessions cannot provide reliable unattended reconciliation; retain console for explicit initiation and navigation. |
| Agentic Operator watches PipelineRuns | Reuses an existing controller deployment | Adds product-specific discovery and Tekton coupling to the shared runtime; keep its contract at submitted runs. |
| Event- or alert-driven producer | Can reuse an existing reliable intake service | Viable if delivery, replay, UID binding, and ownership are demonstrated; no such product contract is established by this ADR. |

## Decision Drivers

Reduce repeated triage without duplicating an existing flow; preserve least privilege and provider choice; bound noise and cost; recover safely after restart; keep optional diagnosis independent of core execution; identify a team that can support the additional component.

## Consequences

### Benefits

- Enrolled teams could receive timely analysis without changing upstream Tekton execution behavior.
- Explicit producer/runtime boundaries allow independent lifecycle and security review.
- Manual-only or reuse of an existing producer remains possible if the additional component is not justified.

### Drawbacks

- A producer adds persistent admission state, operational load, support cost, and a cross-team compatibility dependency.
- Automatic inference can expose workload data and incur cost even when the pipeline failure itself is expected.
- Shared CA trust is broader than the earlier per-server proposal. Namespace RBAC alone also does not deliver source isolation; both require explicit security treatment.
- Retaining deduplication state and protecting reports complicate TTL and source cleanup. Delayed analysis may lose evidence to normal workload pruning.

### Follow-up actions

Record the open acceptance decisions before planning implementation. Separate producer work, runtime security gaps, and console integration; do not turn bounded prototype evidence into a release commitment.

## Dependencies

Requires compatible Agentic Operator/sandbox/MCP delivery, managed endpoint and credential wiring, approved analysis identity and source authorization, receiver-side intake governance, and an authorized report consumer. Shared TLS configuration and toolset enablement are capabilities to validate, not proof that the stronger isolation requirements above are implemented.

## References

- [AgenticRun API and analysis-only workflow](https://github.com/openshift/lightspeed-agentic-operator/blob/main/api/v1alpha1/agenticrun_types.go)
- [AgentTask/Lightspeed prototype evidence and gaps](https://github.com/openshift-pipelines/agenttask-adapter-lightspeed/issues/2)
- [Related PaC follow-up trigger proposal](https://github.com/openshift-pipelines/architecture/pull/14)
- [Earlier PaC investigation ADR, closed for a more scoped proposal](https://github.com/openshift-pipelines/architecture/pull/12)
- [ADR discussion and review](https://github.com/openshift-pipelines/architecture/pull/5)

## Change Log

| Date | Author | Change |
| --- | --- | --- |
| 2026-09-02 | @waveywaves | Initial proposal. |
| 2026-11-17 | @waveywaves | Address reviewer feedback on terminology, alternatives, security boundaries, lifecycle gaps, and current scope. Pending stakeholder review. |
