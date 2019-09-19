---
title: simplify-olm-apis
authors:
  - "@njhale"
reviewers:
  - "@ecordell"
  - "@dmesser"
approvers:
  - TBD
creation-date: 2019-09-05
last-updated: 2019-09-19
status: provisional
see-also:
  - "http://bit.ly/rh-epic_simplify-olm-apis"
---

# simplify-olm-apis

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

This enhancement iterates on OLM APIs to reduce the number of resource types and provide a smaller surface by which to manage an operator's lifecycle. We propose adding a new cluster-scoped resource type to OLM that acts as a single entry point for the installation and management of an operator.

## Motivation

Operator authors perceive OLM/Marketplace v1 APIs as difficult to use. This is thought to stem from three primary sources of complexity:

1. Too many types
2. Redundancies (e.g. OperatorSource and CatalogSource)
3. Effort to munge native k8s resources into OLM resources (e.g. Deployment/RBAC to CSV)

Negative perceptions stunt community adoption and cause a loss of mindshare, while complexity impedes product stability and feature delivery. Reducing OLM's API surface will help to avert these scenarios by making direct interaction with OLM more straightforward. A simplified user experience will encourage operator development and testing.

### Goals

- Define an API resource that aggregates the complete state of an operator
- Define a single API resource that allows an authorized user to:
  - install an operator without ancillary setup (e.g `OperatorGroup` not required)
  - subscribe to installation of operator updates from a catalog
- Be resilient to future changes

- Remain backwards compatible with operators installed by previous versions of OLM
- Retain all of OLM's current features

### Non-Goals

- Define the implementation of an operator bundle
- Describe how bundle images or indexes are pulled, unpacked, or queried
- Deprecate `OperatorSource`
- Customization/configuration of an operator

## Proposal

Introduce a single new cluster scoped API resource that represents an operator as a set of component resources selected with a unique label.

At a High level, an `Operator` spec will specify:

- a `ServiceAccount` to be used by OLM to select and manage components
- a source operator bundle (optional)
- a source package, channel, and bundle index (optional)
- an update policy (optional)

While its status will surface:

- info about the operator, such its name, version, and the APIs it provides and requires
- the label selector used to gather its components
- a set of status enriched references to its components
- top-level conditions that summarize any abnormal state (if any)
- the bundle image or package, channel, bundle index its components were resolved from (if any)
- the packages, channels, and bundle indexes that contain updates for the operator (if any)

An example of what an `Operator` resource would look like:

```yaml
apiVersion: operators.coreos.com/v2alpha1
kind: Operator
metadata:
  name: plumbus
spec:
  install:
    namespace: my-namespace

    serviceAccountRef:
      name: mr-plumbus
      namespace: sa-namespace
  
  updates:
    type: CatalogSource
    catalogSource:
      package: plumbus
      channel: stable
      ref:
        name: community
        namespace: my-ns
      strategy:
        approval: Automatic
        entrypoint: plumbus.v2.0.0

status:
  metadata:
    displayName: Plumbus
    description: Welcome to the exciting world of Plumbus ownership! A Plumbus will aid many things in life, making life easier. With proper maintenance, handling, storage, and urging, Plumbus will provide you with a lifetime of better living and happiness.
    version: 2.0.0-alpha
    apis:
      provides:
      - group: how.theydoit.com
        version: v2alpha1
        kind: Plumbus
        plural: plumbai
        singular: plumbus
      requires:
      - group: manufacturing.how.theydoit.com
        version:
        kind: Grumbo
        plural: grumbos
        singular: grumbo
  
  updates:
    available:
    - name: community
      channel: beta

  conditions:
  - kind: UpdateAvailable
    status: True
    reason: CrossChannelUpdateFound
    message: pivoting between versions
    lastTransitionTime: "2019-09-16T22:26:29Z"

  components:
   matchLabels:
      operators.coreos.com/v2/plumbus: ""
   refs:
    - kind: Subscription
      namespace: operators
      name: plumbus
      uid: d655a13e-d06a-11e9-821f-9a2e3b9d8156
      apiVersion: operators.coreos.com/v1alpha1
      resourceVersion: 109719
    - kind: ClusterServiceVersion
      namespace: operators
      name: plumbus.v2.0.0-alpha
      uid: d70a53b5-d06a-11e9-821f-9a2e3b9d8156
      apiVersion: operators.coreos.com/v1alpha1
      resourceVersion: 109811
      conditions:
      - type: Installing
        status: True
        reason: AllPreconditionsMet
        message: deployment rolling out
        lastTransitionTime: "2019-09-16T22:26:29Z"
    - kind: CustomResourceDefinition
      name: plumbai.how.dotheydoit.com
      uid: d680c7f9-d06a-11e9-821f-9a2e3b9d8156
      apiVersion: apiextensions.k8s.io/v1beta1
      resourceVersion: 75396
    - kind: ClusterRoleBinding
      namespace: operators
      name: rb-9oacj
      uid: d81c24d6-d06a-11e9-821f-9a2e3b9d8156
      apiVersion: rbac.authorization.k8s.io/v1
      resourceVersion: 75438
```

### User Stories

#### As a OLM Admin, I want to

- restrict the resources OLM will apply from a bundle image/index when installing an operator
- restrict the ServiceAccounts a user can install operators with
- restrict the bundle indexes a given user can use to resolve operator bundles

#### As an Operator Installer, I want to

- view the state of an operator by inspecting the status of a single API resource
- deploy an operator by applying its manifests directly
- deploy an operator using a bundle image
- deploy and upgrade an operator by referencing a bundle index

#### As an Operator User, I want to

- view the operators I can use on a cluster

### Implementation Details/Notes/Constraints

#### The `Operator` Resource

The `operators.coreos.com/v2alpha1` API group version will be added to OLM

- alpha versions make no [support guarantees](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions), so resources can be iterated quickly

as well as a command option on `olm-operator` that enables `v1alpha2`

- leave disabled by default

```bash
olm-operator --enable-v2alpha1
```

The cluster-scoped resource, `Operator` will be added to `v2alpha1`

- cluster scoped resources can be listed without granting namespace list, but do require a `ClusterRole`
- namespace scoped resources can specify references to cluster scoped resources in their owner references, which lets us use garbage collection to clean up resources across mutliple namespaces; e.g. `RoleBindings` copied for an `OperatorGroup`

#### Constraining Installation

In a fashion similar `OperatorGroups`, we can constrain the installation of an operator by associating it with a `ServiceAccount`. For an `Operator` resource, the user's desired `ServiceAccount` will be specified by the `spec.install.as` field. The install namespace, where the namespaced contents of an operator will be installed to, is specified by the `install.namespace` field.

```yaml
spec:
  install:
    namespace: my-namespace
    serviceAccountRef:
      name: mr-plumbus
      namespace: sa-namespace
```

OLM will authenticate as this `ServiceAccount` for any requests it makes when installing or otherwise managing the operator. The exception to this rule is when OLM is viewing or mutating resources that affect the Kubernetes API; e.g. `CustomResourceDefinitions` and `APIServices`.

A Cluster Admin can generate `ServiceAccounts` with different installation privileges as needed; e.g. namespace list, pod create, etc. To allow a user to reference a `ServiceAccount`, a Cluster Admin can bind a `Role/ClusterRole` with the `use` custom verb to the user.

This behavior will be enforced by a `ValidatingAdmissionWebhook`. In order to configure serving certs for the webhook

- in OLM's upstream deployment, bootstrap and use [cert-manager](https://docs.cert-manager.io/en/latest/tasks/issuers/index.html) with self-signed certs by default
- in OLM's downstream deployment (OKD), use the [service-ca-operator](https://github.com/openshift/service-ca-operator)
- (optional) at some point support validation/mutating webhooks in OLM and self-host the validating admission server

#### Component Selection

Components can be associated with an `Operator` by a label following the key convention `operators.coreos.com/v2/operator/<operator-name>`. This choice of convention has the following benefits:

- using a unique label key helps avoid collisions
- using a deterministic label key lets users know the exact label used in advance

The resolved label selector for an `Operator` will be surfaced in the `status.components.matchLabels` field of its status.  

```yaml
status:
  components:
    matchLabels:
      operators.coreos.com/v2/operators/my-operator: ""
```

**Note:** *Both namespace and cluster scoped resources are gathered using this label selector. Namespace scoped components are selected across all namespaces.*

Once associated with an `Operator`, a component's reference will be listed in the `status.components.resource` field of that `Operator`. Component references will also be enriched with abnormal status conditions relevant to the operator. These conditions should follow [k8s status condition conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) and in some cases may be copied directly from the component status.

```yaml
status:
  components:
    matchLabels:
        operators.coreos.com/v2/operators/my-operator: ""
    resources:
    - kind: ClusterServiceVersion
      namespace: my-ns
      name: plumbus.v2.0.0-alpha
      uid: d70a53b5-d06a-11e9-821f-9a2e3b9d8156
      apiVersion: operators.coreos.com/v1alpha1
      resourceVersion: 109811
      conditions:
      - type: Installing
        status: True
        reason: AllPreconditionsMet
        message: deployment rolling out
        lastTransitionTime: "2019-09-16T22:26:29Z"
```

Not all components will need to have status conditions surfaced. Initially, it should be sufficient to add conditions for:

- `CustomResourceDefinitions` and `APIServices`
- `Deployments`, `Pods`, and `ReplicaSets`
- `ClusterServiceVersions`, `Subscriptions`, and `InstallPlans`

### Generated Component Selection

For `v1alpha1` and `v1` OLM resource types that result in the generation of new resources, OLM will project labels onto the generated resources when:

- they match the afforementioned label key convention
- an `Operator` resource exists for the label

This means that labels will be copied from:

- `Subscriptions` onto resolved `Subscriptions` and `InstallPlans`
- `InstallPlans` onto `ClusterServiceVersions`, `CustomResourceDefinitions`, RBAC, etc.
- `ClusterServiceVersions` onto `Deployments` and `APIServices`

#### Operator Metdata

The `Operator` resource will surface a `status.metadata` field that

- contains the `displayName`, `description`, and `version` of the operator
- contains the `apis` field which in turn contains the `required` and `provided` apis of the operator

```yaml
status:
  metadata:
    displayName: Plumbus
    description: Welcome to the exciting world of Plumbus ownership! A Plumbus will aid many things in life, making life easier. With proper maintenance, handling, storage, and urging, Plumbus will provide you with a lifetime of better living and happiness.
    version: 2.0.0-alpha
    apis:
      provides:
      - group: how.theydoit.com
        version: v2alpha1
        kind: Plumbus
        plural: plumbai
        singular: plumbus
      requires:
      - group: manufacturing.how.theydoit.com
        version:
        kind: Grumbo
        plural: grumbos
        singular: grumbo
```

OLM will have control logic, such that for each `Operator` resource

- a single `ClusterServiceVersion` in the operator's component selection is picked
  - the _newest_ with respect to the `spec.version` field
- the pick's `displayName`, `description`, and `version` are projected onto the operator's `spec.metadata` field
- the pick's `required` and `provided` APIs are projected onto the operator's `status.metadata.apis` field

#### Status Conditions

To act as an entrypoint for the health of an operator, the `Operator` resource will have a `status.conditions` field that surfaces top-level abnormal status conditions of the operator.

#### Installing From A CatalogSource

OLM will allow a user to subscribe and install changes from `CatalogSources` via the `CatalogSource` variant of the `spec.updates` [union type](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190325-unions.md). A `package`, `channel`, and reference to a `CatalogSource` will be used identify the bundle to install.

```yaml
updates:
  type: CatalogSource
  catalogSource:
    package: my-operator
    channel: stable
    approval: Automatic
    entrypoint: plumbus.v2.0.0
    ref:
      name: community
      namespace: my-ns
```

The optional `approval` field specifies whether `InstallPlans` generated should be automatically or manually installed (defaults to "Automatic"), while he optional `entrypoint` field specifies the name of the first bundle in the package/channel to install.

The `Subscription` resource will be iterated to have a new optional field, `spec.serviceAccountRef`, which will be used by OLM in place of the `ServiceAccount` referenced by `OperatorGroups`.

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: plumbus
  namespace: operators
spec:
  serviceAccountRef:
    name: mr-plumbus
    namespace: sa-namespace
```

When an existing `Subscription` is encountered, OLM will attempt to associate it with an `Operator` resource automatically. Tthe `Subscription` will be associated with an `Operator` resource if its `package`, `channel`, and `CatalogSource` matches the `Operator`'s `spec.updates` field:

- if an `Operator` is already associated with the `Subscription` but the `package`, `channel`, `CatalogSource`, or `ServiceAccount` no longer match, OLM will update the `Subscription`'s `package`, `channel`, `CatalogSource`, and `ServiceAccount` to be in line with the `Operator`.
- if no `Operator` resource is found , OLM will generate a new one with a matching `spec.updates` field of type `CatalogSource`. Additionally, OLM will search for an `OperatorGroup` in the same namespace and
  - if one is found, the `Subscription` will inherit its `ServiceAccount`
  - If zero or more than one is found, then the `Subscription` will inherit the default `ServiceAccount` of its namespace

For each `Operator` that has a `spec.updates` field of type `CatalogSource`, OLM will attempt to generate a `Subscription` if one is not already associated with it. OLM will check if a `Subscription` already exists in the `spec.install.namespace` specifying the same `package` and `CatalogSource`:

- if found, do nothing -- the afforementioned `Subscription` logic will take care of everything
- if not found, generate and associate a new `Subscription` -- the afforementioned `Subscription` logic will take care of the rest

In order to support `OperatorGroup`-less install, OLM's `ClusterServiceVersion` reconciliation will change to rely on the `Operator` resource instead. The `ServiceAccount` used to install resources for a `ClusterServiceVersion` will be inherited from an associated `Operator` resource. If an associated `Operator` resource is not found, OLM will attempt to generate one with steps similar to those described for a `Subscription`, eventually settling on the `default` `ServiceAccount` of the namespace when not found.

In addition to supporting `OperatorGroup`-less install, OLM will clean up copied `ClusterServiceVersions`. When a copy is encountered, OLM will check if the original is associated with an `Operator`:

- if so, add `OwnerReferences` to that `Operator` on all resources in the namespace owned by the copy
- if not, ignore

A top-level status condition, `DivergedFromSource`,  will exist that let's a user know when the components that exist on-cluster diverge from those resolved from an index (the latest `InstallPlan`).

```yaml
conditions:
- type: DivergedFromSource
  status: True
  reason: ComponentsDivergeFromBundleIndex
  message: on-cluster components diverge from bundle index
  lastTransitionTime: "2019-09-16T22:26:29Z"
```

When an update is available for an operator, the `UpdateAvailable` status condition will be surfaced.

```yaml
status:
   conditions:
   - kind: UpdateAvailable
     status: True
     reason: UpdateFoundInAnotherChannel
     message: an upgrade has been found outside the specified channel
     lastTransitionTime: "2019-09-16T22:26:29Z"
```

More detailed information about the package and channel that updates are available in will be specified in the `status.updates.available` field.

```yaml
status:
  updates:
    available:
      - package: my-operator
        channel: alpha
      - package: my-operator
        channel: beta
```

`CatalogSources` will be restricted for use by default. A Cluster Admin can grant a `ServiceAccount` use of a `CatalogSource` by binding the custom `install` verb to it. This can be done for all `CatalogSources` on the cluster by using a `ClusterRoleBinding`, for all in a namespace with a `RoleBinding`, and for a specific `CatalogSource` by specifying its name in a `Role` in the same namespace.

### Installing From A Bundle Image

OLM will enable a user to install a single operator from a bundle image via the `BundleImage` variant of the `spec.source` union type. A fully qualified image (including image registry) will be used to identify the bundle to install.

```yaml
status:
  updates:
    type: BundleImage
    bundleImage: quay.io/my-namespace/my-operator@sha256:123abc...
```

The bundle image will be pulled, unpacked, and applied to the cluster without additional dependency resolution. Top-level status condition `SourceImportError`, with condition reason `BundleImagePullFailed` will be surfaced if any issue is encountered while pulling the bundle image.

```yaml
- type: SourceImportError
  status: True
  reason: BundleImagePullFailed
  message: the bundle image pull has failed
  lastTransitionTime: "2019-09-16T22:26:29Z"
```

Like with the `CatalogSource` variant, an `InstallPlan` will be used to both apply bundle contents and provide a content record for reference. The top-level status condition `DivergedFromSource` will also be used, but with a different condition reason; `ComponentsDivergeFromBundleImage`.

```yaml
conditions:
- type: DivergedFromSource
  status: True
  reason: ComponentsDivergeFromBundleImage
  message: on-cluster components diverge from bundle image
  lastTransitionTime: "2019-09-16T22:26:29Z"
```

### Risks and Mitigations

#### Operator Info Leak

**Problem:** With operators now being represented by a cluster scoped resource, information about installed operators is available outside the scope of the namespace it's deployed into.
**Mitigation:** Make operator get and list privileged operations. `ClusterRoles` can also be created which apply only to specific resources by name.

## Design Details

### Test Plan

Any e2e tests will live in OLM's repo as per convention and will be executed separately from openshift's e2e tests.

The following features should have distinct e2e tests:

- Component selection
- Component status conditions
- Generated resource label propagation
- `Subscription` adoption
- Install from `CatalogSource`
- Install from bundle image
- Direct `ClusterServiceVersion` application
- Copied `ClusterServiceVersion` cleanup

### Graduation Criteria

#### Dev Preview -> Tech Preview

- Users can track operator components with the `Operator` resource
- Users can install an operator from a `CatalogSource` and a bundle image
- `ClusterServiceVersions` can be installed without ancillary setup
- Sufficient test coverage
- End user documentation, relative API stability
- v2alpha1 is bumped to v2beta1
- Gather feedback from users rather than just developers

#### Tech Preview -> GA

- v2beta1 is bumped to v2
- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Feature flags removed
- Available by default

#### Removing a deprecated feature


### Upgrade / Downgrade Strategy

On upgrade, an existing cluster can continue to use OLM as it had before. OLM will infer and generate any new resources required.

Any downgrade procedure would need to take into account that using many different install `ServiceAccounts` in a single namesace is not possible in the previous version of OLM. This means that all `Subscriptions` in a namespace would need to share an `OperatorGroup` with a `ServiceAccount` to drive installation of their resources. This may require that some `Subscriptions` are moved to new namespaces since they could have different scoping requirements. Additionally, the fact that `OperatorGroups` don't support specifying the namespace of a `SerivceAccount` must be taken into account. 

### Version Skew Strategy

All changes are additive, so 

## Implementation History

TODO: section

## Drawbacks

TODO: section

## Alternatives

TODO: section
