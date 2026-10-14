---

## id: ADR-0004

title: OpenShift Builds Integration in OpenShift Pipelines
date: 2026-08-25
status: proposed
authors:

- "@sayan-biswas"
reviewers:
- "@vdemeester"
- "@jkhelil"
- "@waveywaves"
- "@savitaashture"
tags:
- builds
- shipwright
- operator
- pipelines
- tekton

---

# ADR-0004: OpenShift Builds Integration in OpenShift Pipelines

## Context

OpenShift Builds will cease to exist as a separate product and will be made available to users through OpenShift Pipelines. This integration will install the OpenShift Builds components (Shipwright Build and Shared Resources CSI driver) from OpenShift Pipelines Operator when the user installs OpenShift Pipelines from the software catalog.

- **Existing problem**: OpenShift Builds is shipped as a standalone operator that users
must install and manage separately from OpenShift Pipelines, even though Shipwright
Build is built directly on top of Tekton Pipelines APIs.
- **Business goal**: Provide a unified, integrated experience so existing OpenShift
Builds users do not need to install and manage an additional operator.
- **Systems/teams impacted**: Tekton Operator, Shipwright Build, Shared Resource CSI
driver, OpenShift Console (Builds forms), release/packaging.



### Constraints

- The user will to enable `ShipwrightBuild` and `SharedResource` in `TektonConfig`.
- Component controllers will be installed in the `openshift-operators` namespace.
- Components will have a separate set of roles and role bindings.
- Existing forms for Builds in the OpenShift UI should continue to work as-is.



## Decision

`Option A` — Integrate Shipwright Build and Shared Resource using tekton extension.

Two options are under consideration:

- **Option A (recommended)**: Integrate Shipwright Build and Shared Resource as
components of the Tekton Operator, managed through `TektonConfig` using the Tekton
extension framework (`TektonInstallerSet`).
- **Option B**: Bundle the OpenShift Builds operator alongside the OpenShift Pipelines
Operator via the `ClusterServiceVersion` deployments list.

See [Considered Options](#considered-options) for the full comparison.

### Architectural Overview

Shipwright Build uses Tekton Pipelines as its core framework, dynamically creating
`PipelineRun` and `TaskRun` resources to execute a `BuildStrategy` for building
applications (see [https://shipwright.io/docs/introduction](https://shipwright.io/docs/introduction)). Because Shipwright Build
uses Tekton APIs at its core, providing the controller as an extension of Tekton is a
natural fit and follows the Tekton org approach of integrating addon components into the
Tekton ecosystem.

Under the recommended approach (Option A), `shipwright-build` and `shared-resource`
become components under the Tekton Operator, managed by the `TektonConfig` reconciler.
The reconciler creates a `ShipwrightBuild` CR (and a `SharedResource` CR) holding the
component configuration, with an option to enable or disable the Shipwright Build and
Shared Resource deployments from `TektonConfig`. This bypasses the Shipwright Operator and OpenShift Builds operator logic and deploys the release manifests directly using the Tekton extension framework via `TektonInstallerSet`.

### Implementation Details

New additions to the `operator.tekton.dev/v1alpha1` API:

```yaml
# CR to configure and enable Shipwright Build
apiVersion: operator.tekton.dev/v1alpha1
kind: ShipwrightBuild
metadata:
  name: shipwright-build # Created by tekton-config controller
spec:
  enable: true
  # Any other Shipwright Build specific configurations for the future, nothing as of now.
```

```yaml
# CR to configure and enable Shared Resource CSI Driver
apiVersion: operator.tekton.dev/v1alpha1
kind: SharedResource
metadata:
  name: shared-resource # Created by tekton-config controller
spec:
  enable: true
  # Any other Shared Resource CSI driver specific configurations for the future, nothing as of now.
```

Considerations:

- Creating the `ShipwrightBuild` or `SharedResource` CR would by default not install the
components for [Shipwright Build](https://github.com/shipwright-io/build) or the [Shared Resource CSI driver](https://github.com/openshift/csi-driver-shared-resource). They need to be explictly `enabled`.
- Under the default configuration, installing the Tekton Operator would not create these
CRs; instead they will be created by the `TektonConfig` controller when
`ShipwrightBuild` or `SharedResource` is enabled in the `TektonConfig` CR:
  ```yaml
  apiVersion: operator.tekton.dev/v1alpha1
  kind: TektonConfig
  metadata:
    name: config
  spec:
    platforms: # Shipwright Build available for both Kubernetes and OpenShift
      kubernetes:
        shipwrightBuild:
          enabled: true
          # Other configurations from the ShipwrightBuild CR
      openshift:
        shipwrightBuild:
          enabled: true
          # Other configurations from the ShipwrightBuild CR
        sharedResource: # Only available for OpenShift due to a specific dependency
          enabled: true
          # Other configurations from the SharedResource CR
  ```
- There will be a separate controller to reconcile the `ShipwrightBuild` CR and install
the enabled components, in case the user creates the CR themselves.
- Configurations for `ShipwrightBuild` defined in `TektonConfig` should be reflected in
the `ShipwrightBuild` CR created by the `TektonConfig` controller.
- The controller should deploy the required Network Policies for all operands.

**Affected components/systems**: Tekton Operator (`tektoncd/operator`), Shipwright Build,
Shared Resource CSI driver, OpenShift Console Builds forms.

## Security & Compliance

- Shipwright Build and Shared Resource CSI driver will have separate set of roles and role bindings, which must be reviewed for least-privilege RBAC.
- The controller must deploy the required Network Policies for all operands.
- The Shared Resource CSI driver is privileged and OpenShift-specific; its trust
boundary and SCC requirements should be assessed.



## Compatibility & Migration

- Existing OpenShift Builds users will need a migration path from the current
standalone-operator installation to the integrated installation.
- Existing users must manually switch by uninstalling the existing Builds operator and enabling the corresponding components in `TektonConfig`
- A rollback/disable path exists via the `enabled` flags in `TektonConfig` (Option A).



## Testability

- Extensive testing and automated test coverage is required due to the amount of code
being added (especially for Option A, which reimplements the Shipwright and OpenShift
Builds operator logic inside the Tekton Operator).
- A testing strategy spanning unit, integration, and e2e tests should cover enabling and
disabling components through `TektonConfig`, direct CR creation, and reconciliation of
the installed operands.



## Operational Readiness

- Observability for the new reconcilers (`ShipwrightBuild`, `SharedResource`) should be
defined so operators can tell whether the components are installed and healthy.
- New failure modes are introduced: if the Shipwright Build or Shared Resource CSI driver deployment fails, the install strategy will report failure for the component under `TektonConfig` status.



## Scope

- **In-Scope:**
  - Installing/integrating Shipwright Build and Shared Resource as part of the OpenShift Pipelines Operator installation.
  - Enabling/disabling the Build components via `TektonConfig` and/or dedicated CRs.
- **Out-of-Scope:**
  - Upstream Tekton feature changes (these belong in TEPs).
  - OpenShift Builds docs to remain standalone or should be merged.
  - The OpenShift Console UI changes and testing (tracked as a follow-up action).
  - Upstream CLIs (Shipwright Build) integration into OpenShift Pipeline CLI.



## Acceptance Criteria

The decision is considered accepted when:

- [x] Problem, scope, and non-goals are clear
- [x] Alternatives were considered and compared
- [x] Chosen option's trade-offs (not just benefits) are documented
- [x] Backward compatibility / migration path addressed
- [x] Security/RBAC impact assessed
- [ ] Observability and rollback plan defined
- [ ] Required reviewers approved
- [ ] Open concerns resolved or tracked
- [ ] Decision is feasible to implement
- [ ] Documented in internal knowledge base



## Considered Options


| Option                                                                                                                                                                                  | Pros                                                                                                                                                                                                                              | Cons                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Option A: Tekton extension framework** (manage `shipwright-build` and `shared-resource` as Tekton Operator components via `TektonConfig` / `TektonInstallerSet`)                      | Better UX and manageability; follows the Tekton org approach for addon components; unified experience by enabling Builds from `TektonConfig`; no additional release configuration changes; removes the need for a second operator | Requires reimplementing the entire Shipwright and OpenShift Builds operator logic in the Tekton Operator repo (upstream `redhat-openshift-builds/operator` code lives under `internal/` and cannot be vendored, and it uses `manifestival` apply directly instead of `InstallerSet`); extensive testing required |
| **Option B: Bundle operator via CSV** (add `openshift-builds-operator` to the `ClusterServiceVersion` deployments list; Kustomize `builds` component; user creates `OpenShiftBuild` CR) | OpenShift Builds operator installs together with the OpenShift Pipelines Operator; all Build components usable by creating the `OpenShiftBuilds` CR                                                                               | Existing users must manually uninstall the old operator and create the CR; release maintenance is tedious (a new operator shipped with OSP); CRDs have different ownership despite one operator; a failed Builds deployment marks the whole OSP install strategy as failed                                       |
| **Do Nothing**                                                                                                                                                                          | No effort required                                                                                                                                                                                                                | Does not integrate OpenShift Builds into OpenShift Pipelines; users continue to install and manage a separate operator                                                                                                                                                                                           |


Option B reference (`operators.coreos.com/v1alpha1` CSV supports multiple deployments):

```yaml
# Cluster Service Version (Builds deployment added)
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
spec:
  install:
    spec:
      deployments:
        - name: tekton-operator
        - name: tekton-operator-webhook
        - name: openshift-builds-operator
```

For Option B, create a Kustomize component `builds` under the
[Operator Config for OpenShift](https://github.com/tektoncd/operator/tree/main/config/openshift)
holding all manifests required to deploy the OpenShift Builds operator, and include the
manifest under the
[OperatorHub release manifests](https://github.com/tektoncd/operator/blob/main/operatorhub/openshift/manifests/fetch-strategy-release-manifest/kustomization.yaml).
The user then creates the following CR to install the Shipwright Build controllers and
Shared Resource daemonset:

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: OpenShiftBuild
metadata:
  name: cluster
spec:
  sharedResource:
    state: Enabled
  shipwright:
    build:
      state: Enabled
```



## Decision Drivers

Key factors influencing this decision:

- User experience and manageability of the integrated product
- Alignment with the Tekton ecosystem/extension approach
- Complexity and effort required (amount of code to reimplement and maintain)
- Release/packaging maintainability
- Testability and automated test coverage
- Security and RBAC (separate roles, Network Policies, privileged CSI driver)



## Consequences

- Positive outcomes:
  - Shipwright Build and Shared Resource CSI driver is deployed from OpenShift Pipelines Operator, removing the need to install a separate operator.
  - A more integrated and unified user experience for OpenShift Builds users.
  - (Option A) Unified enablement of Build components via `TektonConfig`; no additional
  release configuration changes.
- Risks:
  - (Option A) Large code addition — the full Shipwright and OpenShift Builds operator
  logic must be reimplemented in the Tekton Operator repo.
  - (Option B) Tedious release maintenance, mixed CRD ownership, and a failed Builds
  deployment failing the entire OSP install strategy.
  - Extensive testing effort across both options.
  - Shipwright Build Component might NOT include latest features due to release cycle difference between Shipwright Build and OpenShift Pipelines. This would mean there would be parity between Shipwright Build features offered by OpenShift Builds Operator and the ones offered by this Shipwright Build integration.  



## Dependencies

- Upstream Tekton Operator (`tektoncd/operator`) extension framework and
`TektonInstallerSet`.
- [Shipwright Build](https://github.com/shipwright-io/build).
- [Shared Resource CSI driver](https://github.com/openshift/csi-driver-shared-resource) (OpenShift-specific).
- `redhat-openshift-builds/operator` logic (not vendorable — code under `internal/`).
- OpenShift Console plugin for the Builds forms migration (follow-up).



## References

- [JIRA: Deploy Shipwright and Shared Resource CSI Driver via OpenShift Pipelines](https://redhat.atlassian.net/browse/SRVKP-12244)
- [Shipwright Build introduction](https://shipwright.io/docs/introduction)
- [Tekton Operator — OpenShift config](https://github.com/tektoncd/operator/tree/main/config/openshift)
- [Shared Resource CSI driver](https://github.com/openshift/csi-driver-shared-resource)
- [POC: Add Shipwright Build and Shared Resource extension](https://github.com/tektoncd/operator/pull/4130)



## Follow-up actions

- Define the migration path from the existing installation process to the proposed one.
- Decide whether OpenShift Builds docs remain standalone or are merged with OpenShift
Pipelines docs.
- The OpenShift UI forms for Shipwright currently live under the `Builds` section of the
OpenShift Console; these should be changed to a dynamic console plugin and brought
under OpenShift Pipelines.
- Extend `TektonInstallerSet` to handle `DaemonSet` to report correct `Ready` status for component Shared Resource CSI driver.



## Change Log


| Date       | Author        | Change Summary                           |
| ---------- | ------------- | ---------------------------------------- |
| 2026-08-25 | @sayan-biswas | Created initial ADR                      |
| 2026-10-14 | @sayan-biswas | Reformatted to the standard ADR template |


---

