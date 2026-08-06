# 0003. Pipelines MultiKueue Plugin

Date: 2026-08-06

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

The MultiCluster feature was delivered with OpenShift Pipelines release v1.22.0. This release is in Tech Preview and currently requires many manual steps before anyone can start using MultiCluster.

Cluster management is the biggest challenge with this release: it requires manual setup on each spoke cluster, plus additional manual setup on the hub cluster for every spoke cluster added.

This initiative aims to eliminate all manual steps currently required to enable the MultiCluster feature — including installing required operators, and creating custom resources, secrets, and service accounts. The plugin should take full responsibility for setting up the Hub and Spoke clusters, with no user intervention required.

## Constraints

- The Fleet Admin must provide the list of spoke clusters, with proper authentication for each.

## Design

This will be a dedicated Kubernetes controller watching cluster resources on the Hub cluster.

### Watchers

The controller watches the following resources.

#### For ACM-Enabled Users

- **ManagedClusters**: When a user imports or detaches a cluster from the hub, the controller reconciles resources on both the Hub and the affected cluster. The full list of reconciled resources is in the [Resources Being Reconciled](#resources-being-reconciled) section below.
- **ManagedServiceAccounts and Managed Secrets**: These are watched for token rotation events from ACM. When ACM rotates a spoke cluster's token, the controller updates the corresponding token in the MultiKueue secrets.

#### For Non-ACM-Enabled Users
- **Spoke Secrets**: These secrets will follow a specific pattern. User will create secrets for each spoke cluster and controller will onboard the spoke cluster to fleet.

### Reconcilers

The controller operates in two modes:

- **Hub cluster**: The controller watches resources on the hub cluster directly, and the main reconciler reconciles resources there.
- **Spoke clusters**: Each spoke cluster requires its own kube client. On every reconcile event, the controller creates (or reuses) a kube client for the target spoke cluster and reconciles resources on it.

#### Resources Being Reconciled

In addition to installing the operator on the hub and spoke clusters, the controller reconciles the following resources:

**On Hub:**
- MultiKueue Cluster
- MultiKueue Config
- ManagedServiceAccount
- Kueue Secrets

**On Spoke:**
- ClusterQueue
- LocalQueue
- TektonConfig (to enable Kueue on spoke clusters)

#### Other Reconciliation Features

**Upgrading Operators**

Since the controller installs operators on the Hub and Spoke clusters, those operators must be managed exclusively by the controller. The controller will:
- Add an owner reference to all resources it creates (not to resources it only updates).
- Upgrade operators it manages when a newer version is required.

**Managing Namespaces**

- TBD

## Decision

_State the decision concisely in one or two sentences. Add subsections if more detail is needed._

## Consequences

- Should increase adoption of MultiCluster Pipelines, as customers no longer need to follow a lengthy manual setup guide to configure multicluster on their fleet.
- Should make testing easier, as developers and QE benefit from a faster setup process.

### Benefits

- **Environment Agnostic**: Natively supports both OpenShift Plus (ACM) customers and vanilla Kubernetes/Tekton users.
- **Graceful Degradation**: The same controller handles both scenarios — it simply swaps its cluster-discovery mechanism (ACM/OCM APIs vs. standard Secrets) based on the environment.

### Drawbacks

- Adds an additional component to manage.

### Follow-up actions

- _TBD_