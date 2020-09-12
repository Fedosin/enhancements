---
title: out-of-tree-provider-support
authors:
  - "@Danil-Grigorev"
reviewers:
  - "@enxebre"
  - "@elmiko"
  - "@mfedosin"
approvers:
  - "@enxebre"
  - "@elmiko"
  - TBD
creation-date: 2020-08-31
last-updated: 2020-09-12
status: implementable
see-also:
  - "/enhancements/cloud-controller-manager/openstack-cloud-controller-manager.md"  
replaces:
superseded-by:
---

# Add support for out-of-tree cloud provider integration

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement proposal describes steps needed to add support for out-of-tree providers in cloud providers currently supported by Openshift. This includes: AWS, Azure, GCP, vSphere, IBM. Openstack provider proposal is located at it's dedicated [document](https://github.com/openshift/enhancements/tree/master/enhancements/cloud-controller-manager/openstack-cloud-controller-manager.md).

## Motivation

The Kubernetes community is currently moving towards exclusion of the in-tree cloud-provider code from the core Kubernetes repository, and instead introducing out-of-tree provider support, making them independent from the core Kubernetes release cycle, and simplifying the process of development and e2e testing for those plugins. Previously each cloud-provider was located in the [`legacy-cloud-providers`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/legacy-cloud-providers) and now the support and main development is done in the dedicated cloud-provider [repositories](https://github.com/kubernetes?q=cloud-provider-). They plan to remove the in-tree support completely in the future releases, approximetly in [v1.21](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600).

### Goals

- Prepare openshift components to accomodate the out-of-tree plugins and remove support for the in-tree implementation.
- Describe an approach to select an `external` cloud provider installation for the cloud.
- Provide the means to allow generic `cloud-controller` component registration via the provider specific upstream implemetation.

### Non-Goals

- Force immediate exclusion of in-tree support for currently used providers, as their out-of-tree counterparts are added.
- Add support for CSI and other out-of-tree implementations of cluster storage interfaces.
- Add support for running multiple cloud configurations in a single cluster.

## Proposal

Add an out-of-tree support based on current IPI with minor adjustments, and make a seamless transition for all currently supported providers.

This change will be gradually executed on all suported cloud providers: AWS, Azure, GCP, vSphere and IBM.

### User Stories

As a cloud developer, I’d like to improve support for cloud features for openshift

#### Story 1:

We’d like to deploy control-plane node on a spot instance, to allowing customers to get additional cost reduction on their clusters. In order to introduce and test this feature, we have to implement this in the kubernetes repository, first, then wait for the next release to get a “free” rebase on the feature. We are also bound to the kubernetes release cycles, making it hard to achieve the goal in case the core Kubernetes is already going through the feature freeze. Starting downstream is not an option, as the implementation could be rejected upstream or gain less attention then needs to quickly get it into the release, diverging our fork from the source.

#### Story 2:

As an openshift developer I’m assigned a release blocker BZ regarding upstream implementation. Proving the value for merging this and making the bug a release blocker upstream requires careful communication with upstream maintainers, and explaining the value to the people whom are not necessarily involved into internal details, so their opinion could differ and delay merging an important fix.

#### Story 3:

We desire to extend our team responsibilities on upstream cloud providers in the future and gain weight in promoting features into upstream. Getting such for kubernetes repository is harder for us and maintainers, as this would mean giving weight for approving and merging features for the parts of the kubernetes project, not necessarily laying under our responsibilities and scope of knowledge.

#### Story 4:

We’d like to discuss technical details related to a specific cloud in a SIG meeting with people who are also involved into development in this domain, and that way gain useful insights into the cloud infrastructure, improve the overall quality of our features, stay on top of the new features, and improve the relations with maintainers outside of our company, which nevertheless share with us a common goal.


### Implementation Details

The `external` `cloud-controller-manager` implementation will be hosted within openshift repository in https://github.com/openshift/cluster-cloud-controller-manager-operator. That repository will manage `CCM` provisioning for metioned cloud providers + `Openstack` out-of-tree provider implementation described in the [document](https://github.com/openshift/enhancements/tree/master/enhancements/cloud-controller-manager/openstack-cloud-controller-manager.md)

`Cloud-controller` will be provisioned in the `openshift-cloud-controller-manager` namespace.

Preparation steps, common for all providers:

- Add support for recognition of the `external` option in [library-go](https://github.com/openshift/library-go/blob/master/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go) which will disable config parsing and add support for `external` field in the `Infrastructure` resource.
- Port the templates of `cloud-controller` from upstream repositories into proposed https://github.com/openshift/cluster-cloud-controller-manager-operator repository. Those templates will belong to the newly created `openshift-cloud-controller-manager` namespace and be managed with an operator.
- Operator will be deployed in the `openshift-cloud-controller-manager-operator` namespace.

Each provider implementation will need to do the following minimal set of actions in order to make out-of-tree implemetation work:

- Use `external` flag in the `kube-controller-manager` pod instead of the previously used provider. Remove the `cloud-config` option from the template.
- Use `external` flag in the `kubelet` pod. Remove the `cloud-config` option from the template.
- Provision a `cloud-controller-manager` `DaemonSet` in the `openshift-cloud-controller-manager` namespace.

#### API resource changes

In order to achieve desired cluster configuration with `external` cloud components, we need to add a new field into `Infrastructure` [resource](https://github.com/openshift/api/blob/c2f7aea5d89ee87d7510b50bdb08dde9028a53ef/config/v1/types_infrastructure.go#L11). With the addition, a `status.platformStatus.external` field will determine the architectural change in the cluster configuration. Other components will be able to consume the configuration same way as before, only with a minor change of the resource processing.

```go
type PlatformStatus struct {
  ...
	// External is a value used in infrastucture automaton to determine component-specific
	// behavior for the PlatformType value. Components depending on this variable will
	// ignore cloud specific detailes associated with the given platform type,
	// and will concider the value of the infrastructure to be "external" - essentially
	// disabling any further evaluation.
	//
	// This will allow to delegate task of processing platform specific bits
	// to another component.
	// +optional
  External bool `json:"external"`
  ...
```

To ensure a smooth transition from the `in-tree` selection, `library-go` [implementation](https://github.com/openshift/library-go/blob/master/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go) will include an additional argument for the `cloudProviderObserver` structure, which will allow to preserve initial configuration strategy, until the new `observeExternal` flag will be set to `false`:

```go
type cloudProviderObserver struct {
	targetNamespaceName     string
	cloudProviderNamePath   []string
	cloudProviderConfigPath []string
	observeExternal         bool
}
```

The `observeExternal` field will determine, if `status.platformStatus.external` field from the `Infrastructure` resource will be taken in account in establishing needed flags for a Kubernetes binary, such as some `cloud-controller-manager`, `kube-controller-manager`, etc. It will make the observer type to ignore content of the `cloud-config` file, and force the provider to be set as `external` if the option is not selected.

#### Kubelet

Openshift manages `kubelet` configuration with `machine-config-operator`. The actual `cloud-config` arguments are passed externally from the `Infrastructure` resource on https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357, where the support for `external` cloud provider will be added. For each platform with `external` configuration, the `cloud-config` flag will not be set.

#### Kube-controller-manager

This configuration is managed by the `cluster-kube-controller-manager-operator` code and will need to integrate the `library-go` changes, described above in the operator [configuration](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85).

#### Kube-api-server

No changes for the current configuration are needed

### Operator resource management strategy

Every cloud manages it's configuration differently, but share a common overall structure.

1. `cloud-controller-manager` and related resources are deployed under one namespace.
2. Each pod (`ccm`, `cloud-node-manager`, etc.) supports deployment with a `DaemonSet`.
3. The pods are expected to run on `Nodes` marked as a control-plane with annotation:
```yaml
    node-role.kubernetes.io/master: ""
```
4. The pods are sharing common command line arguments with the `kcm` pods, such as `--cloud-provider` and `--cloud-config`.

Proposed operator will not diverge from the upstream resource structuring:

- Static infrastructure resources creation, such as namespaces or service accounts will be delegated to the [CVO](https://github.com/openshift/cluster-version-operator).
- Operator will populate, create and manage the cloud-specific set of `DaemonSets`, based on the selected `platform` in the `Infrastructure` resource.
- The `--cloud-provider` and the `--cloud-config` arguments will be populated using existing `library-go` based implementation from the [cluster-kube-controller-manager-operator](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85).

#### Examples for the cloud-controller-manager configurations

##### AWS

Main repository: https://github.com/kubernetes/cloud-provider-aws/
Sample manifests: https://github.com/kubernetes/cloud-provider-aws/tree/master/manifests

To try external cloud-controller-manager, simply run:
```bash
oc apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-aws/master/manifests/rbac.yaml  
oc apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-aws/master/manifests/aws-cloud-controller-manager-daemonset.yaml
```

##### Azure

Main repository: https://github.com/kubernetes-sigs/cloud-provider-azure
Sample manifests: https://github.com/kubernetes-sigs/cloud-provider-azure/tree/master/examples/out-of-tree

##### GCP

Main repository: https://github.com/kubernetes/cloud-provider-gcp
Sample manifests: https://github.com/kubernetes/cloud-provider-gcp/tree/master/deploy

##### vSphere

Main repository: https://github.com/kubernetes/cloud-provider-vsphere
Sample manifests: https://github.com/kubernetes/cloud-provider-vsphere/tree/master/manifests/controller-manager

## Upgrade/Downgrade strategy

While a set of `cloud-controller-manager` pods is running under a `DaemonSet` provisioned by the operator, aquiring leadership over cloud specific control loops in a HA openshift deployment will be managed by a cross-namespace leader-election migration.

Upstream [proposal](https://github.com/kubernetes/enhancements/blob/473d6a094f07e399079d1ce4698d6e52ddcc1567/keps/sig-cloud-provider/20190422-cloud-controller-manager-migration.md#motivation) describes the main implementaion details for this component. Their proposed implementation for cross-namespace leadership delegation would help Openshift to preserve HA during upgrage from in-tree to the out-of-tree.

In case the cluster does not require to be HA, without integrating this feature an upgrade under bare CVO management will lead to two simultanious replicas of `cloud-controllers` running at the same time at some moment.

### Test Plan

- Each provider at the time of migration should have a working CI implemetation, which will assist in testing the provisioning with out-of-tree support enabled.
- Ensure that transition between the in-tree and the out-of-tree will be handled by the `Infrastracture` `status.platformStatus.external` field being set.
- Ensure the in-tree and the out-of-tree providers could co-exist in the cluster if this is supported.
- Ensure that upgrade from a release running on the in-tree provider only to the out-of-tree succeeds.

## Other notes

##### Removing a deprecated feature

Upstream is currently planning to remove support for the in-tree providers with OCP-4.8 (kubernetes 1.21) - based on the slack [conversation](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600). The enchancement [describes](https://github.com/kubernetes/enhancements/blob/473d6a094f07e399079d1ce4698d6e52ddcc1567/keps/sig-cloud-provider/20190125-removing-in-tree-providers.md) steps towards this process. Our goal is to have a working alternative for the in-tree providers ready to be used on-par with old implementation befor the upstream release will remove this feature from core Kubernetes repository.

## Timeline

Follow the Kubernetes community discussions and implementation/releases

### Kubernetes 1.18

- vSphere support graduaded to beta: https://github.com/kubernetes/enhancements/issues/670

### Kubernetes 1.19

- Azure support goes into beta: https://github.com/kubernetes/enhancements/issues/667

## Infrastructure Needed

### New github projects:

Forks for out-of-tree provider repositories:
- [AWS](https://github.com/kubernetes/cloud-provider-aws)
- [GCP](https://github.com/kubernetes/cloud-provider-gcp)
- [Azure](https://github.com/kubernetes-sigs/cloud-provider-azure)
- [Vsphere](https://github.com/kubernetes-sigs/cloud-provider-vsphere)

Mandatory operator repository:
- [CCM-Operator](https://github.com/openshift/cluster-cloud-controller-manager-operator)
