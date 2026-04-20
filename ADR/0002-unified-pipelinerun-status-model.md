# 2. Unified PipelineRun Status Model for Tekton Ecosystem Controllers

Date: 2026-04-20

## Status

Proposed

## Context

A `PipelineRun` in Tekton passes through the hands of multiple controllers after the core
pipeline controller marks it complete. Today, each of these controllers records its outcome
exclusively in `metadata.annotations`:

| Controller       | Annotation key(s)                                              | Meaning                                   |
|------------------|----------------------------------------------------------------|-------------------------------------------|
| Tekton Chains    | `chains.tekton.dev/signed`                                     | Signing / attestation outcome             |
| Tekton Results   | `results.tekton.dev/record`, `results.tekton.dev/result`       | Archival record URI                       |
| Pipelines as Code| `pipelinesascode.tekton.dev/state`                             | VCS commit-status posted (GitHub / GitLab)|

This design creates compounding friction:

1. **Dual-path observation.** Dependent systems must watch both `.status.conditions` and
   `.metadata.annotations`, implement two separate reconciliation code paths, and handle
   potential race conditions between them.

2. **No composite readiness signal.** There is no single field that means "the pipeline
   succeeded _and_ every downstream controller has finished its work." A system that must
   not proceed until Chains has signed a run has to inspect annotations independently.

3. **Multi-cluster sync gaps.** In hub-and-spoke topologies (e.g. Kueue, OCM, or
   ArgoCD-based fan-out), status sync is a first-class primitive; annotation propagation
   is not. Spoke-to-hub replication therefore requires bespoke annotation copy logic
   alongside the status sync, duplicating effort and introducing drift.

4. **Failure attribution ambiguity (the RCA trigger).** When PAC posts a failed VCS
   commit-status because of a transient 503 from GitHub, it writes
   `pipelinesascode.tekton.dev/state=failed` into annotations. Nothing in the current
   model separates "the pipeline itself failed" from "a post-processing controller
   encountered an error." Dependent systems that treat any `failed` annotation as a
   pipeline failure will produce false negatives on genuinely successful runs.

### Kubernetes ecosystem precedents

- **`status.conditions` (KEP-1027 / API conventions):** Kubernetes codifies conditions as
  typed, machine-readable status signals with a `reason`, `message`, `lastTransitionTime`,
  and an `observedGeneration`. The convention is well-understood by controller authors and
  tooling (e.g. `kubectl wait --for=condition=`).
- **`managedFields` (Server-Side Apply):** Kubernetes tracks _which field manager owns
  which field_ inside the object itself. This is the closest prior art for "multiple
  controllers writing into the same object in a structured, non-colliding way."
- **Aggregated ClusterRole / RBAC:** The aggregation label pattern lets independent objects
  contribute rules into a parent without the parent's controller knowing about the children
  — a useful model for decentralised contribution with a central view.
- **`Gateway API` / status.parents:** Multiple route-binding controllers each write into
  a typed sub-slice of `.status`, scoped by a controller identity key, rather than
  sharing a flat annotation namespace.

---

## Decision

Introduce a **structured extension block** within `PipelineRun.status` — provisionally
named `status.processingControllers` — that ecosystem controllers can write to using
Server-Side Apply (SSA). Add a single **composite condition**
(`ProcessingComplete`) to `status.conditions` that aggregates the overall readiness
across all registered controllers.

The core PipelineRun controller **does not own** the new fields and **requires no
modification**; the extension is additive and fully backward-compatible.

---

### Proposed Schema

#### `PipelineRun.status` extension (CRD structural schema)

```yaml
# Illustrative OpenAPI v3 schema fragment — added to the existing PipelineRun CRD
status:
  processingControllers:
    type: object
    description: >
      Keyed map of per-controller processing states contributed via Server-Side Apply.
      Each key is the controller's SSA field-manager name.
    x-kubernetes-preserve-unknown-fields: false
    additionalProperties:
      type: object
      required: [controllerName, phase]
      properties:
        controllerName:
          type: string
          description: Human-readable controller identifier.
        phase:
          type: string
          enum: [Pending, Running, Succeeded, Failed, Skipped]
        reason:
          type: string
          description: Machine-readable CamelCase reason code.
        message:
          type: string
          description: Human-readable detail; may include transient error context.
        lastTransitionTime:
          type: string
          format: date-time
        observedGeneration:
          type: integer
          format: int64
        url:
          type: string
          description: >
            Optional deep-link for UIs (e.g. Chains transparency-log entry,
            Results archival record, PAC VCS commit-status page).
        retryCount:
          type: integer
          description: Number of reconciliation attempts so far (useful for transient failures).
```

**Concrete example** — a fully-processed run:

```yaml
status:
  conditions:
    - type: Succeeded
      status: "True"
      reason: Succeeded
      lastTransitionTime: "2026-04-20T10:00:00Z"
    - type: ProcessingComplete      # <-- new composite condition
      status: "True"
      reason: AllControllersSucceeded
      message: "chains-controller, results-controller, pac-controller all succeeded."
      lastTransitionTime: "2026-04-20T10:05:30Z"
      observedGeneration: 3
  processingControllers:            # <-- new extension block
    chains-controller:
      controllerName: Tekton Chains
      phase: Succeeded
      reason: Signed
      message: "Attestation stored at https://rekor.sigstore.dev/..."
      lastTransitionTime: "2026-04-20T10:01:15Z"
      url: "https://rekor.sigstore.dev/api/v1/log/entries?logIndex=12345"
      retryCount: 0
    results-controller:
      controllerName: Tekton Results
      phase: Succeeded
      reason: Archived
      message: "Record stored at results.tekton.dev/v1alpha2/..."
      lastTransitionTime: "2026-04-20T10:02:40Z"
      url: "https://results.tekton.dev/v1alpha2/parents/default/results/abc/records/xyz"
      retryCount: 0
    pac-controller:
      controllerName: Pipelines as Code
      phase: Succeeded
      reason: VCSStatusPosted
      message: "Commit status 'success' posted to github.com/org/repo@sha."
      lastTransitionTime: "2026-04-20T10:05:28Z"
      url: "https://github.com/org/repo/commit/abc123"
      retryCount: 0
```

**Failure isolation example** — PAC hits a transient 503:

```yaml
status:
  conditions:
    - type: Succeeded
      status: "True"           # pipeline outcome unchanged
      reason: Succeeded
    - type: ProcessingComplete
      status: "False"          # composite gate is open — downstream should wait / retry
      reason: ControllerFailed
      message: "pac-controller is in phase Failed (VCSPostFailed): upstream 503."
  processingControllers:
    chains-controller:
      phase: Succeeded
      reason: Signed
      # ...
    pac-controller:
      phase: Failed
      reason: VCSPostFailed
      message: "GitHub returned HTTP 503. Retry 3/5."
      retryCount: 3
```

The core pipeline `Succeeded=True` condition is never touched; only the composite
`ProcessingComplete` condition and the PAC-owned slice of `processingControllers` reflect
the failure.

---

### Controller Interaction Model

```
┌────────────────────────────────────────────────────────────────────────┐
│                          Kubernetes API Server                         │
│                                                                        │
│  PipelineRun                                                           │
│  ├─ metadata.annotations   (legacy — kept for backward compat)        │
│  └─ status                                                             │
│     ├─ conditions[Succeeded]         ← owned by core PR controller    │
│     ├─ conditions[ProcessingComplete]← owned by an aggregator *       │
│     └─ processingControllers                                           │
│        ├─ chains-controller          ← owned by Chains via SSA        │
│        ├─ results-controller         ← owned by Results via SSA       │
│        └─ pac-controller             ← owned by PAC via SSA           │
└───────────────┬──────────────────────────────────────────────────────┘
                │  watch PipelineRun (status.conditions[Succeeded]=True)
       ┌────────┼──────────────────────────────────┐
       ▼        ▼                                  ▼
 Chains      Results                              PAC
 controller  controller                           controller
       │        │                                  │
       │  SSA-patch processingControllers[self]    │
       └────────┴──────────────────────────────────┘
                │
                ▼
       Aggregator controller  (new, lightweight)
       Watches processingControllers, computes
       ProcessingComplete condition, SSA-patches it.
```

> \* **Aggregator controller** — a small, stateless controller (or a webhook) that watches
> `PipelineRun` objects and recomputes the `ProcessingComplete` condition whenever any
> `processingControllers` entry changes. It owns only `conditions[ProcessingComplete]` via
> SSA and has no opinion about pipeline correctness.

#### Write protocol — Server-Side Apply

Each ecosystem controller patches its own slice using SSA with a dedicated `fieldManager`
name (e.g. `chains-controller`, `pac-controller`). SSA guarantees:

- No controller can accidentally overwrite another controller's slice.
- Conflict detection is automatic; a controller that tries to claim a field already owned
  by another manager will receive a conflict error.
- `managedFields` in the object records exactly which manager owns which field, making
  ownership auditable with `kubectl get pipelinerun -o json | jq .metadata.managedFields`.

#### RBAC

```yaml
# Each controller's ServiceAccount receives only the minimum necessary permissions
rules:
  - apiGroups: ["tekton.dev"]
    resources: ["pipelineruns/status"]
    verbs: ["patch", "get", "watch", "list"]
```

Separation of status subresource RBAC from the main resource means no controller can
modify `spec`, labels, or the core `status.conditions[Succeeded]` field.

---

### Backward Compatibility and Migration

The annotation-based contracts are **not removed** in the initial rollout.

| Phase | Annotations | `processingControllers` | Notes |
|-------|-------------|------------------------|-------|
| 1 — Dual-write | Written (unchanged) | Written (new) | Both present; zero breaking change for existing consumers. |
| 2 — Annotation deprecation | Written + deprecation notice | Written | `kubectl.kubernetes.io/last-applied-configuration`-style deprecation warnings in controller logs; docs updated. |
| 3 — Annotation removal | Removed | Written | Only after all known consumers have migrated; gated behind a feature flag. |

Feature-flag gating (`tekton.dev/processingControllers: enabled`) in each controller's
`ConfigMap` allows operators to opt individual controllers into Phase 1 before committing
the whole fleet.

---

### Composite `ProcessingComplete` Condition

The aggregator follows standard KEP-1027 condition semantics:

| `status` | `reason` | Meaning |
|----------|----------|---------|
| `Unknown` | `Pending` | No ecosystem controller has reported yet. |
| `Unknown` | `Running` | At least one controller is still `Running` or `Pending`. |
| `True` | `AllControllersSucceeded` | Every registered controller reached `Succeeded` or `Skipped`. |
| `False` | `ControllerFailed` | At least one controller is in `Failed` phase. |
| `False` | `Timeout` | A controller did not report within the configured deadline. |

**Registration** — controllers declare themselves in a `ConfigMap` (or a lightweight CRD
`PipelineRunProcessor`) watched by the aggregator. Only declared controllers are waited
upon; unknown keys in `processingControllers` are ignored for the composite gate.

```yaml
# ConfigMap-based registry (simple, no new CRD required for MVP)
apiVersion: v1
kind: ConfigMap
metadata:
  name: pipelinerun-processors
  namespace: tekton-pipelines
data:
  controllers: |
    - name: chains-controller
      timeout: 5m
    - name: results-controller
      timeout: 2m
    - name: pac-controller
      timeout: 3m
```

---

### Multi-Cluster Implications

With this model, spoke-to-hub synchronisation requires **no bespoke annotation handling**:

- Tools that sync `status` subresources (Kueue's `PodGroupStatus`, OCM `ManifestWork`
  status feedback, or a custom `StatusSync` controller) automatically carry
  `processingControllers` and the `ProcessingComplete` condition, because they are part of
  the canonical `status` object.
- The hub cluster can evaluate `ProcessingComplete` without knowledge of which controllers
  ran on the spoke, or what annotations they would have written.
- Annotation propagation logic — currently bespoke in most hub-spoke Tekton setups — is
  eliminated for controllers that have migrated to Phase 2 or Phase 3.

---

### Failure Isolation

The model enforces strict scoping so that a downstream controller failure cannot change
the pipeline's own outcome:

1. **Owned fields are disjoint.** The core controller owns `conditions[Succeeded]`.
   Ecosystem controllers own only their own `processingControllers[<self>]` entry.
   SSA conflicts prevent any controller from touching a field it does not own.

2. **The composite condition is a separate type.** `ProcessingComplete=False` is
   semantically distinct from `Succeeded=False`. A system that only cares about pipeline
   correctness watches `Succeeded`; a system that needs full post-processing confirmation
   watches `ProcessingComplete`.

3. **`retryCount` + `phase` distinguish transient from terminal failures.** A PAC entry
   in `phase: Failed, retryCount: 3` communicates "still retrying" context without
   escalating to the pipeline outcome. The aggregator can optionally keep
   `ProcessingComplete=Unknown/Running` while retries remain below the configured limit,
   only flipping to `False` once the controller marks the entry terminal.

4. **Cause attribution is explicit.** The `reason` and `message` fields on the failing
   entry (e.g. `reason: VCSPostFailed, message: "GitHub returned HTTP 503"`) give
   operators the RCA signal directly in the status, without cross-referencing controller
   logs or annotation diffs.

---

## Consequences

### Benefits

- **Single source of truth.** Dependent systems watch `status.conditions[ProcessingComplete]`
  instead of polling annotations across multiple keys.
- **Ownership clarity.** SSA `managedFields` makes field ownership auditable with standard
  tooling.
- **No core controller changes required.** The core PipelineRun controller is entirely
  unmodified; adoption risk is scoped to ecosystem controllers.
- **Multi-cluster sync is trivially correct.** Status sync tools carry the new fields
  automatically.
- **Failure isolation by design.** A downstream controller failure cannot alter the
  pipeline's `Succeeded` condition.
- **Extensible.** Future controllers (e.g. a policy controller, a cost-attribution
  controller) contribute a new entry without modifying any schema or the aggregator's
  core logic; only the `ConfigMap` registry needs an entry.

### Drawbacks

- **Aggregator controller is a new operational dependency.** It must be deployed,
  monitored, and kept available. Its failure would stall the `ProcessingComplete`
  condition. Mitigation: make it a lightweight deployment with a liveness probe; scope its
  RBAC to status-only patches.
- **Dual-write period increases API server write traffic.** During Phase 1, each
  ecosystem controller writes both an annotation and a status patch per PipelineRun.
  At typical pipeline volumes this is negligible, but high-throughput clusters should
  measure before committing to the rollout timeline.
- **CRD schema evolution requires care.** Adding `processingControllers` as
  `additionalProperties` with a structural schema is straightforward, but further schema
  changes to the per-controller object require a CRD version bump and conversion webhook
  if the shape changes incompatibly.
- **Controller registration via ConfigMap is eventually-consistent.** If the registry
  `ConfigMap` is updated after a PipelineRun starts, the aggregator may compute an
  incomplete `ProcessingComplete` condition for in-flight runs. Mitigation: snapshot the
  registry at PipelineRun completion time (record it in an annotation or a status field)
  and use that snapshot for aggregation.

### Follow-up actions

- [ ] Open a Tekton Enhancement Proposal (TEP) to formalise `processingControllers` as an
      upstream API addition to `tekton.dev/v1`.
- [ ] Implement a prototype aggregator controller in a dedicated repository under
      `tektoncd/` and iterate on the registration model.
- [ ] Add SSA-based status patching to Tekton Chains, Tekton Results, and PAC behind
      feature flags.
- [ ] Validate the multi-cluster scenario against a Kueue + OpenShift hub-spoke topology.
- [ ] Define a conformance test suite that verifies correct failure isolation (i.e.
      `Succeeded=True` is never modified by an ecosystem controller).
- [ ] Revisit annotation deprecation timeline once at least two controllers have shipped
      Phase 1 dual-write in a stable release.
