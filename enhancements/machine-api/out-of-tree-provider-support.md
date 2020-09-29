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
last-updated: 2020-09-29
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
- Concider support for running multiple cloud configurations in a single cluster. Build architecture with this concideration in mind.

### Non-Goals

- Force immediate exclusion of in-tree support for currently used providers, as their out-of-tree counterparts are added.
- Add support for CSI and other out-of-tree implementations of cluster storage interfaces.

## Proposal

Add an out-of-tree support based on current IPI with minor adjustments, and make a seamless transition for all currently supported providers.

This change will be gradually executed on all suported cloud providers: AWS, Azure, GCP and vSphere. The change could help inclusion for other cloud providers, such as [Yandex](https://github.com/flant/yandex-cloud-controller-manager) and IBM `CCMs`, if this will be concidered a goal.

### User Stories

#### Story 1:

As a cloud developer, I’d like to improve support for cloud features for openshift. I'd like to develop and test new cloud providers with do not ship with default kubernetes distiribution, and require support for external cloud controller manager.

#### Story 2:

As a developer I'd like to develop, build and release fixes independently from the kubernetes core, and assume they will first land upstream and then will be carried over into openshift destribution with less effort.

#### Story 3:

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

#### Existing resource changes

##### Development

In order to achieve desired cluster configuration with `external` cloud components, we need to add an annotation on the `Infrastructure` [resource](https://github.com/openshift/api/blob/c2f7aea5d89ee87d7510b50bdb08dde9028a53ef/config/v1/types_infrastructure.go#L11). The annotation will determine the architectural change in the cluster configuration. Other components will be able to consume the configuration same way as before, and will decide which flags will be set, if this annotation would be found.

Proposed annotation: `infrastructure.openshift.io/external-cloud-provider: true`.

The annoation will be a temporary measure to simlify the operator development, and enable testing the transition in CI. Once all providers will become out-of-tree, the annotation will be safe to remove.

To ensure a smooth transition from the `in-tree` selection, `library-go` [implementation](https://github.com/openshift/library-go/blob/master/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go) will include an additional argument for the `cloudProviderObserver` structure, which will allow to preserve initial configuration strategy, until the new `observeExternal` flag will be set to `false`:

```go
type cloudProviderObserver struct {
	targetNamespaceName     string
	cloudProviderNamePath   []string
	cloudProviderConfigPath []string
	observeExternal         bool
}
```

The `observeExternal` field will determine, if annotation `external-cloud-provider` is present on the `Infrastructure` resource, and will be taken in account while establishing needed flags for a Kubernetes binaries. It will make the observer type to ignore content of the `cloud-config` file, and force the provider to be set as `external` if the option is not selected. The `kube-controller-manager`, `kube-api-server` and `kubelet` flag will be set as:

```bash
# No cloud-config option
--cloud-provider=external
```

#### Final adjustments

Once the provider is ready to be published with an extenal `CCM` support, the code parts for in-tree enablement will be removed on [these](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85) lines.

#### Kubelet

Openshift manages `kubelet` configuration with `machine-config-operator`. The actual `cloud-config` arguments are set externally, based on the `Infrastructure` resource values on https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357, where the support for `external` cloud provider will be added. For each platform with `external` configuration, the `cloud-config` flag will not be set. 

Kubelet is tightly coupled with `CCMs`, due to its [usage](https://github.com/kubernetes/kubernetes/blame/323f34858de18b862d43c40b2cced65ad8e24052/pkg/kubelet/cloudresource/cloud_request_manager.go#L98-L102) of cloud-provider interfaces, such as `Instances` and `NodeAdresses`. Whenever there is a need to bootstrap any `Node`, `kubelet` will collect missing information about the instance IP addresses, type and zone. This information could later be used by `kube-scheduler`.

During this procedure all uninitialized `Nodes` in the cluster are [tainted](https://github.com/kubernetes/kubernetes/blob/c53ce48d48372c30053a9b67c6d1dee237b5cd69/pkg/kubelet/kubelet_node_status.go#L307-L311) with `node.cloudprovider.kubernetes.io/uninitialized: true`. This taint is removed after the [initialization](https://github.com/kubernetes/kubernetes/blob/46d522d0c8e94edb6d29606a28785e35259badd7/staging/src/k8s.io/cloud-provider/controllers/node/node_controller.go#L374-L376) by the cloud-provider.

#### Kube-controller-manager

This configuration is managed by the `cluster-kube-controller-manager-operator` code and will need to integrate the `library-go` changes, described above in the operator [configuration](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85).

#### Kube-api-server

This component requires same [configuration](https://github.com/openshift/cluster-kube-apiserver-operator/blob/d54e109b333e4db52b681a6979c706b1bdd3a778/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L133) changes as `kcm`.

### Operator resource management strategy

Every cloud manages it's configuration differently, but share a common overall structure.

1. `cloud-controller-manager` and related resources are deployed under one namespace.
2. Each pod (`CCM`, `cloud-node-manager`, etc.) supports deployment with a `Deployment` resource.
3. The pods are expected to run on `Nodes` marked as a control-plane with annotation:
```yaml
    node-role.kubernetes.io/master: ""
```
4. The `CCM` pods are sharing common command line arguments with the `kcm` pods, such as `--cloud-provider` and `--cloud-config`.

Proposed operator will not diverge from the upstream resource structuring:

- Static infrastructure resources creation, such as namespaces or service accounts will be delegated to the [CVO](https://github.com/openshift/cluster-version-operator).
- Operator will populate, create and manage the cloud-specific set of `Deployments`, based on the selected `platform` in the `Infrastructure` resource.
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

Azure separated the `CCM` and the "cloud-node-manager" (`CNM`) into two binaries: https://github.com/kubernetes-sigs/cloud-provider-azure/tree/master/cmd.

##### GCP

Main repository: https://github.com/kubernetes/cloud-provider-gcp
Sample manifests: https://github.com/kubernetes/cloud-provider-gcp/tree/master/deploy

##### vSphere

Main repository: https://github.com/kubernetes/cloud-provider-vsphere
Sample manifests: https://github.com/kubernetes/cloud-provider-vsphere/tree/master/manifests/controller-manager

Vsphere introduced an additional service to expose internal API server: https://github.com/kubernetes/cloud-provider-vsphere/blob/master/manifests/controller-manager/vsphere-cloud-controller-manager-ds.yaml#L64

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

### Upstream 

Follow the Kubernetes community discussions and implementation/releases

Kubernetes 1.18:
- vSphere support graduaded to beta: https://github.com/kubernetes/enhancements/issues/670

Kubernetes 1.19
- Azure support goes into beta: https://github.com/kubernetes/enhancements/issues/667

### Openshift

Release 4.7:
- Migrate openstack provider on the out-of-tree implementation.

Release 4.8:
- Migrate vSphere, AWS, Azure and GCP providers on out-of-tree.

## Infrastructure Needed

### New github projects:

Forks for out-of-tree provider repositories:
- [AWS](https://github.com/kubernetes/cloud-provider-aws)
- [GCP](https://github.com/kubernetes/cloud-provider-gcp)
- [Azure](https://github.com/kubernetes-sigs/cloud-provider-azure)
- [Vsphere](https://github.com/kubernetes/cloud-provider-vsphere)

Mandatory operator repository:
- [CCM-Operator](https://github.com/openshift/cluster-cloud-controller-manager-operator)
