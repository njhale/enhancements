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

## Proposal

The following section outlines a desired goal post for OLM's APIs and behavior without prescribing explicit steps to reach it. The "how to get there" is discussed in the [implementation section](#implementation-details/notes/constraints)

### Representing an Operator

Introduce a single new cluster scoped API resource that represents an operator as a set of component resources selected with a unique label.

At a High level, an `Operator` spec will specify:

- a `ServiceAccount` to be used by OLM to select and manage components
- a source operator bundle (optional)
- a source package, channel, and bundle index (optional)
- a customization to apply to the bundle before installation (optional)
- an update policy (optional)

While its status will surface:

- info about the operator, such its name, version, and the APIs it provides and requires
- the label selector used to gather its components
- a set of status enriched references to its components
- top-level conditions that summarize any abnormal state (if any)
- the bundle image or package, channel, bundle index its components were resolved from (if any)
- the packages, channels, and bundle indexes that contain updates for the operator (if any)

Define the `Operator` type as:

```golang

```

An example of what an `Operator` resource would look like:

```yaml
apiVersion: operators.coreos.com/v2alpha1
kind: Operator
metadata:
  name: plumbus
spec:
  install:
    as:
      name: mr-plumbus
      namespace: my-namespace
      kind: ServiceAccount

    updateStrategy:
      approval: Automatic

    customizations:
      - type: NamespaceMapping
        namespaceMapping:
          from: default
          to: operators
  
  source:
    type: BundleIndex
    bundleIndex:
      package: plumbus
      channel: stable
      ref:
        name: community
        namespace: my-ns
        apiVersion: operators.coreos.com/v1alpha1
        kind: CatalogSource

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
  
   source:
    availableUpdates:
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

### Constraining Installation

In a fashion similar `OperatorGroups`, we can constrain the installation of an operator by associating it with a `ServiceAccount`. For an `Operator` resource, the user's desired `ServiceAccount` will be specified by the `spec.install.as` field.

```yaml
spec:
  install:
    as:
      name: mr-plumbus
      namespace: my-namespace
      kind: ServiceAccount
```

OLM will authenticate as this `ServiceAccount` for any requests it makes when installing or otherwise managing the operator. The exception to this rule is when OLM is viewing or mutating resources that affect the Kubernetes API; e.g. `CustomResourceDefinitions` and `APIServices`.

A Cluster Admin can generate `ServiceAccounts` with different installation privileges as needed; e.g. namespace list, pod create, etc. To allow a user to reference a `ServiceAccount`, a Cluster Admin can bind a `Role/ClusterRole` with the `use` custom verb to the user. This behavior will be enforced by a `ValidatingAdmissionWebhook`.

### Selecting Components

Components can be associated to an `Operator` with a label following key convention `operators.coreos.com/v2/operator/<operator-name>`. The resolved label selector for an `Operator` will be surfaced in the `status.components.matchLabels` field of its status.  

```yaml
status:
  components:
    matchLabels:
      operators.coreos.com/v2/operators/my-operator: ""
```

> Note: Both namespace and cluster scoped resources are gathered using this label selector. Namepsace scoped components are selected across all namespaces.

Once associated with an `Operator`, a component's reference will be listed in the `status.components.resource` field of that `Operator`. Component references will also be enriched with abnormal status conditions relevant to the operator.

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

Not all resources will need to have status conditions surfaced. Initially, it should be sufficient to add conditions for:

- `CustomResourceDefinitions` and `APIServices`
- `Deployments`, `Pods`, and `ReplicaSets`
- `ClusterServiceVersions`, `Subscriptions`, and `InstallPlans`

For `v1alpha1` and `v1` OLM resource types that result in the generation of new resources (like `Subscriptions`, `InstallPlans`, and `ClusterServiceVersions`), OLM will project labels onto the generated resources when:

- they match the afforementioned label key convention
- an `Operator` resource exists for the label

### Status Conditions

To act as an entrypoint for the health of an operator, the `Operator` resource will have a `status.conditions` field that surfaces top-level abnormal status conditions of the operator.

### Installing From Bundle Indexes

OLM will allow a user to subscribe and installing changes from bundle indexes via the `BundleIndex` variant of the `spec.source` union type. A package, channel, and reference to a `CatalogSource` will be used identify the bundle to install.

```yaml
source:
  type: BundleIndex
  bundleIndex:
    package: my-operator
    channel: stable
    ref:
      name: community
      namespace: my-ns
      apiVersion: operators.coreos.com/v1alpha1
      kind: CatalogSource
```

For the most part, dependency resolution will remain largely unchanged, with resolved resources being written out to an `InstallPlan`. A top-level status condition, `DivergedFromSource`,  will exist that let's a user know when the components that exist on-cluster diverge from those resolved from an index (the latest `InstallPlan`).

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

More detailed information about the package and channel that updates are available in will be specified in the `status.source.availableUpdates` field.

```yaml
status:
  source:
    availableUpdates:
      - package: my-operator
        channel: alpha
      - package: my-operator
        channel: beta
```

Bundle indexes in the form of `CatalogSources` will be restricted for use by default. A Cluster Admin can grant a `ServiceAccount` use of a `CatalogSource` by binding the custom `install` verb to it. This can be done for all `CatalogSources` on the cluster by using a `ClusterRoleBinding`, for all in a namespace with a `RoleBinding`, and for a specific `CatalogSource` by specifying its name in a `Role` in the same namespace.

### Installing From Bundle Images

OLM will enable a user to install a single operator from a bundle image via the `BundleImage` variant of the `spec.source` union type. A fully qualified image (including image registry) will be used to identify the bundle to install.

```yaml
source:
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

Like with the `BundleIndex` variant, an `InstallPlan` will be used to both apply bundle contents and provide a content record for reference. The top-level status condition `DivergedFromSource` will also be used, but with a different condition reason; `ComponentsDivergeFromBundleImage`.

```yaml
conditions:
- type: DivergedFromSource
  status: True
  reason: ComponentsDivergeFromBundleImage
  message: on-cluster components diverge from bundle image
  lastTransitionTime: "2019-09-16T22:26:29Z"
```

### Customizing Bundles

A user can tell OLM to customize the content installed from a bundle image or index by specifying the `spec.install.customizations` field of an `Operator` resource. This field will be a list of a union type and will let users specify multiple different types of customizations at once. Customizations will be applied to bundles resolved from indexes or images before they are applied to a cluster.

The primary customization type, `NamespaceMapping`, will allow a user to map the namespaces used in a bundle to existing namespaces on a cluster. This lets an operator installation span multiple namespaces while still giving the installer an opportunity to pick what those namespaces are.

```yaml
spec:
  install:
    customizations:
      - type: NamespaceMapping
        namespaceMapping:
          from: default
          to: operators
```

Other customization types may be added later to provide an even more flexible customization options, like [kustomize](https://github.com/kubernetes-sigs/kustomize).

Not all operators are written to tolerate namespace changes (or other customizations). To provide an escape hatch, OLM will not allow customizations to operator bundles not marked as customizable by the bundle author.

> Note: The marking mechanism is dependent on how bundle packaging is implemented. This could take the form of an annotation on a bundle image, or a field in the operator's metadata.

An `Operator` attempting to customize a bundle that doesn't support customization will surface the `UnsupportedCustomization` top-level status condition.

```yaml
conditions:
- type: UnsupportedCustomization
  status: True
  reason: BundleNotCustomizable
  message: bundle has not been marked as customizable by the author
  lastTransitionTime: "2019-09-16T22:26:29Z"
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

This section outlines addative steps to acheive the goal laid out in the [proposal section](#proposal).

#### Define the initial `Operator` resource

Add the `operators.coreos.com/v2alpha1` API group version to OLM

- alpha versions make no [support guarantees](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions), so resources can be iterated quickly

Add a command option to `olm-operator` that enables `v1alpha2`

- leave disabled by default

```bash
olm-operator --enable-v2alpha1
```

Add the cluster scoped `Operator` resource to the `v2alpha1`

- cluster scoped resources can be listed without granting namespace list, but do require a `ClusterRole`
- namespace scoped resources can specify references to cluster scoped resources in their owner references, which lets us use garbage collection to clean up resources across mutliple namespaces; e.g. `RoleBindings` copied for an `OperatorGroup`

#### Enable user delegation

Define, as part of the `Operator` resource, a `spec.install.as` field that:

- lets a user pick the `ServiceAccount` to use when managing the operator

Add a validating admission server to OLM and use a `ValidatingAdmissionWebhook` to include it in the aggregation layer.

In order to configure serving certs

- in OLM's upstream deployment, bootstrap and use [cert-manager](https://docs.cert-manager.io/en/latest/tasks/issuers/index.html) with self-signed certs by default
- in OLM's downstream deployment (OKD), use the [service-ca-operator](https://github.com/openshift/service-ca-operator)
- (optional) at some point support validation/mutating webhooks in OLM and self-host the validating admission server

Add admission logic that, for any `Operator` resource,  rejects any create/modify requests for users that do not have the `use` role bound to the specified `ServiceAccount`.

- `use` must be bound with a `ClusterRole`

#### Enable component selection

Define a label convention for selecting resources that compose an operator

- `operators.coreos.com/v2/operators/<name>: ""`; where `name` is `metadata.name`
- using a unique label key helps avoid collisions
- using a deterministic label key lets users know the exact label used in advance

Define, as part of the `Operator` resource, a `status.components` field that surfaces:

- `matchLabels`: label selector used to list operator components
- `resources`: references to resources selected by `matchLabels`

Add OLM controller logic such that, for each `Operator` resource:

- `matchLabels` is resolved from the label convention and stored in the `status.components.matchLabels` field

Add OLM controller logic such that, for each `Operator` resource:

- OLM authenticates as the specified s`ServiceAccount`
- cluster-scoped resources are selected using `status.components.matchLabels`
- namespace-scoped resources are selected __across all namespaces__ using `status.components.matchLabels`
- the union of both selections is stored in `status.components.refs`
- kinds selected in both cases may be predefined or queried from discovery

#### Enable generated component selection

Add OLM controller logic, such that for each `Operator` resource:

- OLM resources matching `status.components.matchLabels` have the matching labels projected onto all related generated resources
  - from `Subscriptions` onto resolved `Subscriptions` and `InstallPlans`
  - from `InstallPlans` onto `ClusterServiceVersions`, `CustomResourceDefinitions`, RBAC, etc.
  - from `ClusterServiceVersions` onto `Deployments` and `APIServices`

#### Enable operator metadata

Define, as part of the `Operator` resource, a `status.metadata` field that

- contains the `displayName`, `description`, and `version` of the operator
- contains the `apis` field which in turn contains the `required` and `provided` apis of the operator

Add OLM controller logic, such that for each `Operator` resource

- a single CSV in the operator's component selection is picked
  - the _newest_ with respect to the `spec.version` field
- the pick's `displayName`, `description`, and `version` are projected onto the operator's `spec.metadata` field
- the pick's `required` and `provided` APIs are projected onto the operator's `status.metadata.apis` field

#### Enable component status conditions

Define, as part of the `Operator` resource, a `status.components.refs[*].conditions` field that

- enriches each element with conditions representing the state of the referenced component
- should be relevant in the context of an operator
- need not be surfaced for all component kinds
- should follow [k8s status condition conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties)
- may be copied directly from the component

Phase-in OLM controller logic such that, for each `Operator`:

- component conditions are copied directly from
  - `Subscriptions`
  - `InstallPlans`
  - `ClusterServiceVersions`
  - `Deployments`

#### Enable operator status conditions

Define, as part of the `Operator` resource, a `status.conditions` field that

- surfaces status conditions which describe the overall abnormal state of the operator
- should follow [k8s status condition conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties)

#### Enable installing from a bundle index

Define a `spec.source` field that

- is a union type
- contains a `type` field that acts as a discriminator between variants

Define `BundleIndex` as a new value of `spec.source.type` that indicates the optional `spec.source.bundleIndex` is present.

Define `spec.source.bundleIndex` to contain the following fields:

- `package`: package associated with the operator to resolve
- `channel`: channel in the package to resolve from
- `ref`: object reference to the resource representing the index. Initially, the only accepted `kind` should be `CatalogSource`

Add control logic to OLM such that for each `Subscription` found, OLM

- extracts the `ServiceAccount` from an `OperatorGroup` in the `Subscription`'s namespace
  - if more than one `OperatorGroup` is found, skip all `Subscriptions` in the namespace
  - if none are found, skip all `Subscriptions` in the namespace
  - if exactly one is found, take the `ServiceAccount` specified
- attempts to find an `Operator` resource with a source matching its package, channel, and index (`CatalogSource`) and a `ServiceAccount` matching the one extracted from the `OperatorGroup`
  - if found, adds `OwnerReferences` to the `Operator` on all namespaced resources generated by the `Subscription` and deletes it
  - if not found, generates an `Operator` with a matching `source`

Update `ClusterServiceVersion` reconciliation to use `Operator` resources instead of `OperatorGroups`

- ensure that two `Operators` providing the same API is not allowed

Update OLM's install/dependency resolution logic

- resolve from `Operator` resources instead of `Subscriptions`
  - if the `ServiceAccount` __does not have__ the `install` verb bound to the `CatalogSource`
    - add status condition `SourceNotAuthorized` with reason `ServiceAccountNotAuthorized` to the `Operator` resource
    - prevent further installtion/dependency resolution
  - if the `ServiceAccount` __does have__ the `install` verb bound to the `CatalogSource`
    - remove the `SourceNotAuthorized` status condition
- resources applied from `InstallPlans` directly, without modifying `metadata.namespace`
- all `InstallPlans` are automatic by default
- set appropriate status conditions on index pull failure and resolution failure

Add logic to clean up any _copied_ `ClusterServiceVersions`

- when an `Operator` exists for the origin `ClusterServiceVersion`
  - add `OwnerReferences` to the `Operator` resource on any resources owned by the copied `ClusterServiceVersion`
  - delete the copied `ClusterServiceVersion`

#### Enable installing from a bundle image

Define `BundleImage` as a new value of `spec.source.type` that indicates the optional `spec.source.bundleImage` is present.

Define `spec.source.bundleImage` as a reference to an image

Add control logic to OLM such that for each `Operator` resource, OLM

- attempts to pull the image referenced in `spec.source.bundleImage`
  - if unsuccessful, add the `SourceImportError` status condition to the `Operator` resource with reason `BundleImagePullFailed`
- applies the resolved resources to the cluster without resolution

- If you specify `install.namespace`, OLM applies all namespaced stuff in that namespace from a bundle
- If we want to support arbitrary apply namespace in the future, we have a new field for that `customizations`
- If the operator name is placeholder, then we require an `install.namespace`

### Risks and Mitigations

TODO: section

## Design Details

### Test Plan

TODO: section

### Graduation Criteria

TODO: section

#### Examples

TODO: section

##### Dev Preview -> Tech Preview

TODO: section

##### Tech Preview -> GA

TODO: section

##### Removing a deprecated feature

TODO: section

### Upgrade / Downgrade Strategy

TODO: section

### Version Skew Strategy

TODO: section

## Implementation History

TODO: section

## Drawbacks

TODO: section

## Alternatives

TODO: section
