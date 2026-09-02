# 0005. Automatic Agentic Diagnosis for Failed PipelineRuns

Date: 2026-09-02

## Status

Proposed

## Context

OpenShift Pipelines users diagnose failed `PipelineRun`s by correlating Tekton status, events, Pod status, and logs. Enrolled teams need analysis to begin automatically while retaining manual diagnosis.

The generic Agentic Operator reconciles `AgenticRun`s but does not discover Tekton failures. Tekton controllers and the console are unsuitable background producers.

Automatic analysis consumes model capacity and may send workload data to the configured provider. The managed MCP certificate chains to a private CA absent from the sandbox system trust store. Global CA installation or disabled verification would widen trust. [OLS-3857](https://redhat.atlassian.net/browse/OLS-3857) tracks isolated per-server trust.

## Decision

OpenShift Pipelines will add a standalone, Pipelines-owned controller that creates analysis-only `AgenticRun`s for eligible `PipelineRun` failures.

Automatic diagnosis requires both:

1. `TektonConfig.spec.agenticDiagnosis.enabled=true`, disabled by default; and
2. namespace label `pipelines.openshift.io/automatic-diagnosis=enabled`.

The field also sets validated cluster and namespace concurrency, log-byte, and timeout limits. Provider and MCP configuration remain separate. A `PipelineRun` may opt out with annotation `pipelines.openshift.io/automatic-diagnosis=disabled`.

Failures and timeouts trigger diagnosis; cancellations and graceful stops do not. The controller creates one deterministic, source-owned `AgenticRun` per `PipelineRun` UID in the source namespace. It neither mutates the source nor maintains a private queue. Reconciliation retries deferred work, and manual diagnosis remains available.

Each analysis receives a short-lived, same-namespace identity. The source name and UID are bound through trusted run configuration, not a prompt or model-controlled argument. MCP tools verify child relationships before reading related `TaskRun`s, Pods, events, or bounded logs. Unrelated workloads, Secrets, Pod execution, cross-namespace access, and all mutations are denied. Diagnosis visibility must be no broader than `pods/log` access. Automatic remediation is excluded.

For each server with `caFile`, the sandbox creates an isolated TLS context from system roots plus that server's PEM CA bundle. It preserves hostname and chain verification, never changes global trust, and never uses `verify=false`. Invalid trust fails before token transmission, and one server's CA is not trusted by another. [OLS-3594](https://redhat.atlassian.net/browse/OLS-3594), [OLS-3858](https://redhat.atlassian.net/browse/OLS-3858), and [OLS-3859](https://redhat.atlassian.net/browse/OLS-3859) provide injection, identity, and read-only tools.

Missing dependencies fail closed and set `AgenticDiagnosisReady=False` without making healthy core Pipelines NotReady.

## Consequences

### Benefits

- Failures receive timely, Kubernetes-native diagnosis without coupling Tekton or the generic Agentic Operator to product-specific discovery.
- Two-party opt-in, source-bound tools, scoped identity, and isolated TLS trust limit data and authorization exposure.
- Deterministic reconciliation provides deduplication, backpressure, and cleanup through existing Kubernetes state.

### Drawbacks

- Pipelines gains another supported component and a coordinated Lightspeed compatibility dependency.
- Automatic analysis consumes model capacity, may incur cost, and may send bounded logs to an external configured provider.
- Concurrency limits or unavailable dependencies can delay diagnosis; deleting the source also deletes its diagnosis.

### Follow-up actions

- Define `TektonConfig` defaults, validation, status, and enrollment documentation.
- Implement, package, threat-model, and audit the controller and source-bound MCP authorization.
- Complete [OLS-3857](https://redhat.atlassian.net/browse/OLS-3857), [OLS-3594](https://redhat.atlassian.net/browse/OLS-3594), [OLS-3858](https://redhat.atlassian.net/browse/OLS-3858), and [OLS-3859](https://redhat.atlassian.net/browse/OLS-3859).
- Add released-stack tests for triggers, deduplication, limits, dependency failure, unrelated-resource denial, CA isolation and rotation, RBAC, result visibility, and cleanup.
- Track delivery in [SRVKP-12191](https://redhat.atlassian.net/browse/SRVKP-12191).
