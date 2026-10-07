# 0004. OpenShift Builds Integration in OpenShift Pipelines

Date: 2026-08-25

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

OpenShift Builds will cease to exist as separate product and will made available to user through OpenShift Pipelines.
This integration will install OpenShift Builds operator along with OpenShift Pipelines Operator from the user installs OpenShift Pipelines from software catalog.

### Constraints

- This will only install the OpenShift Builds Operator and user would need to create a `OpenShiftBuilds` CR to install the Build controllers.
- The OpenShift Builds operator will be installed in `openshift-operators` namespace
- The OpenShift Builds controllers will be installed in `openshift-builds` namespace
- OpenShift Builds will have separate set or roles and role bindings
- Existing forms for Builds in OpenShift UI should work as it is.


### Proposal

This solution that provides better user experience and manageability for the integration but requires more code changes in the tekton operator repository.
This is uses tekton extension to add `shipwright-build` and `shared-resource` as component under tekton operator, managed by `TektonConfig`
reconciler to create a `ShihpwrightBuild` CR with that hold configurations for Shipwright Build and an option to Enable or Disable Shipwright Build and Shared Resource deployment from `TektonConfig`. This would bypass all the code
of the `Shipwright Operatpr` and `OpenShift Builds` logic operator and deploy the manifests directly using Tekton extension framework, using `TektonInstallerSet`. Obviously this would require all the logic from the `Shipwright` and `OpenShift Builds` operator to be implemented in tekton operator repo.

Addition to operator/v1aplha1 API:
```yaml
# CR to configure and enable Shipwright Build
apiVersion: operator.tekton.dev/v1alpha1
kind: ShipwrightBuild
metadata:
  name: shipwright-build # Created by tekton-config controller
spec:
  enable: true
  # Any other Shipwright Build specific configurations for future, nothing as of now.
```

```yaml
# CR to configure and enable Shared Resource CSI Driver
apiVersion: operator.tekton.dev/v1alpha1
kind: SharedResource
metadata:
  name: shared-resource # Created by tekton-config controller
spec:
  enable: true
  # Any other Shared Resource CSI driver specific configurations for future, nothing as of now.
```

#### Considerations:
- Creating this `ShipwrightBuild` or `SharedResource` CR would by default not install the components for 
  [Shipwright Build](https://github.com/shipwright-io/build), or `[Shared Resource CSI driver](https://github.com/openshift/csi-driver-shared-resource)`.
- Under default configuration, installing `Tekton Operator` would not create this CR, instead it will be created by
  `TektonConfig` controller on enabling `ShipwrightBuild` or `SharedResource` in the `TektonConfig` CR.
  ```yaml
  apiVersion: operator.tekton.dev/v1alpha1
  kind: TektonConfig
  metadata:
    name: config
  spec:
    platforms: # Shipwright build available for both Kubernetes and OpenShift Platform
      kubernetes: 
        shipwrightBuild:
          enabled: true 
          # Other configurations from the ShipwrightBuild CR       
      openshift:
        shipwrightBuild:
          enabled: true 
          # Other configurations from the ShipwrightBuild CR 
        sharedResource: # Only available for OpenShift platform due to specific dependency
          enabled: true
          # Other configurations from the SharedResource CR 
  ```
- There will be separate controller to reconcile `ShipwrightBuild` CR and install the enabled components, in case user creates the CR by themselves.
- Configurations for `ShipwrightBuild` defined in `TektonConfig` should be reflected in the `ShipwrightBuild` CR created by `TektonConfig` controller.
- The controller should deploy the required Network Policies for all operands.

### Benefits
- Shipwright Build uses tekton pipelines as core framework on which it creates `PipelineRun` and `TaskRun` dynamically to executes a `BuildStrategy` to build specific applications. More details here https://shipwright.io/docs/introduction. Since Shipwright Build uses tekton APIs at its core, so it makes to provide the controller from an extension of tekton. In OpenShift platform this give the current OpenShift Builds users a more integrated user experience and removes the necessity to install another operator. 
- OpenShift Builds operator will be installed as a component of Tekton Operator and all the components for OpenShift Builds like Shipwright, Shared Resources can in installed by enabling the corresponding component in `TektonConfig` and also by creating the `ShipwrightBuild` CR.
- This approach follows the Tekton org approach of integrating addon components in tekton ecosystem.
- Unified user experience across all components by enabling OpenShift Builds from `TektonConfig`.
- No additional changes required in Release configurations.

### Drawbacks

- The whole code of `Shipwright` and `OpenShiftBuilds` operator need to implemented here. Vendoring any code from upstream `redhat-openshift-builds/operator` is not possible because the code is present in `internal` directory and is not allowed to import.
  Also, the code uses `manifestival` apply directly, which is not the approach in tekton extension framework, instead it uses `InstallerSet`.
- Extensive testing and automated test coverage is required due to the amount of code addition.

### Alternate Proposal

`operators.coreos.com/v1alpha1`, Cluster Service Version defines deployments as a list, which allows to specify multiple deployments.

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

Create Kustomize component `builds` under [Operator Config for OpenShift](https://github.com/tektoncd/operator/tree/main/config/openshift). The component will hold all the manifest required to deploy OpenShift Builds operator.
Include the manifest under [OperatorHub release manifests](https://github.com/tektoncd/operator/blob/main/operatorhub/openshift/manifests/fetch-strategy-release-manifest/kustomization.yaml)

User creates the following CR to install Shipwright Builds controllers and Shared resources daemonset.
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

### Benefits

- OpenShift Builds operator will be installed with OpenShift Pipelines Operator. Then all the components for OpenShift Builds like Shipwright, Shared Resources can in installed and used by creating the `OpenShiftBuilds` CR.


### Drawbacks

- Existing OpenShift Builds users should manually switch by uninstalling existing builds operator and creating the `OpenShiftBuilds` custom resource.
- Maintaining release would be tedious, as there's new operator being shipped with the OpenShift Pipelines Operator.
- OpenShift Builds Operator CRDs and OpenShift Pipelines Operator CRDs will have different ownership even though they are deployed using the same operator.
- If the OpenShift Builds deployment fails, install strategy will show failed for all OpenShift Pipelines Operator.

## Decision

TBD

## Consequences

OpenShift Builds operator will be deployed along with OpenShift Pipelines Operator. After that user can optionally create `OpenShiftBuilds` CR to install Shipwright Builds and Shared Resources


### Follow-up actions

- Migration path from existing installing process to this proposed installation process
- OpenShift Builds docs to exists as standalone or merged with OpenShift Pipelines
- The OpenShift UI forms for Shipwright exists under `Builds` section under OpenShift Console. It should be changed to dynamic console plugin and brought under OpenShift Pipelines. 
