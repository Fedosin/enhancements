---
title: neat-enhancement-idea
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
last-updated: 2020-09-01
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

## Open Questions

1) Should we running `cloud-controller-manager` as:
- static `Pod` managed from the `Node` filesystem.
- `Deployment` backing up the `Pod`.
- `ReplicaSet` to run a `Pod` on each master node.

## Summary

This enhancement proposal describes steps needed to add support for out-of-tree providers in cloud providers currently supported by Openshift. This includes: AWS, Azure, GCP, vSphere, IBM. Openstack provider proposal is located at it's dedicated document.

## Motivation

Upstream is currently moving towards exclusion of in-tree cloud-provider code from `k/k` repository, and instead introducing out-of-tree provider support, making them independent from `k/k` releases, and simplifying the process of development and e2e testing for those plugins. Previously each cloud-provider was located in the [`legacy-cloud-providers`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/legacy-cloud-providers) and now the support and main development is done in the dedicated cloud-provider [repositories](https://github.com/kubernetes?q=cloud-provider-). Upstream is planning to remove in-tree support completely in the future releases, approximetly in [v1.21](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600).

### Goals

- Prepare openshift components to accomodate the out-of-tree plugins and remove support for in-tree implementation.
- Add an installer option to select an`external` cloud provider.
- Deploy cluster infrastructure accordingly to selected `external` option for cloud provider, and allow `cloud-controller` component registration via the provider specific upstream implemetation.

### Non-Goals

- Force immediate exclusion of in-tree support for currently used providers, as their out-of-tree counterparts are added.
- Add support for CSI and other out-of-tree implementations of cluster storage interfaces.
- Add support for running multiple cloud configurations in a single cluster.

## Proposal

Add an out-of-tree support based on current IPI, and make a seamless transition for all currently supported providers.

This change will be gradually executed on all suported cloud providers: AWS, Azure, GCP, vSphere and IBM.

### User Stories

As a cloud developer, I’d like to improve support for cloud features for openshift

#### Story 1:

We’d like to deploy control-plane node on a spot instance, to allowing customers to get additional cost reduction on their clusters. In order to introduce and test this feature, we have to implement this in the kubernetes repository, first, then wait for the next release to get a “free” rebase on the feature. We are also bound to the kubernetes release cycles, making it hard to achieve the goal in case the upstream is already going through the feature freeze. Starting downstream is not an option, as the implementation could be rejected upstream or gain less attention then needs to quickly get it into the release, diverging our fork from the source.

#### Story 2:

As an openshift developer I’m assigned a release blocker BZ regarding upstream implementation. Proving the value for merging this and making the bug a release blocker upstream requires careful communication with upstream maintainers, and explaining the value to the people whom are not necessarily involved into internal details, so their opinion could differ and delay merging an important fix.

#### Story 3:

We desire to extend our team responsibilities on upstream cloud providers in the future and gain weight in promoting features into upstream. Getting such for kubernetes repository is harder for us and maintainers, as this would mean giving weight for approving and merging features for the parts of the kubernetes project, not necessarily laying under our responsibilities and scope of knowledge.

#### Story 4:

We’d like to discuss technical details related to a specific cloud in a SIG meeting with people who are also involved into development in this domain, and that way gain useful insights into the cloud infrastructure, improve the overall quality of our features, stay on top of the new features, and improve the relations with maintainers outside of our company, which nevertheless share with us a common goal.


### Implementation Details/Notes/Constraints

The `external CCM` implementation will be hosted within openshift repository in https://github.com/cluster-cloud-controller-manager-operator. That repository will manage `CCM` provisioning for metioned cloud providers + `Openstack` out-of-tree provider implementation.

`Cloud-controller` will be provisioned in the `openshift-cloud-controller` namespace.

Preparation steps, common for all providers:
- Add support for recognition of the `external` option in library-go: https://github.com/openshift/library-go/blob/master/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go#L154 which will disable config parsing and add `external-cloud-config` option in the `Infrastructure` resource, which will be only used for external providers.
- Port the templates of `cloud-controller` from upstream repositories into proposed https://github.com/openshift/cluster-cloud-controller-manager-operator repository. Those templates will belong to newly created `openshift-cloud-controller` namespace and be managed with an operator.

Each provider implementation will need to do the following minimal set of actions in order to make out-of-tree implemetation work:
- Add `external` flag to `kube-controller-manager` pod instead of the previously used provider. Remove the `cloud-config` option from the template.
- Add `external` flag to `kubelet`. Remove the `cloud-config` option from the template.
- Deploy a `cloud-controller` pod in the `openshift-cloud-controller` namespace.

## Design Details

### Kubelet

Openshift manages `kubelet` configuration with `machine-config-operator`. The actual `cloud-config` arguments are passed externally from the `Infrastructure` resource on https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357, where the support for `external` cloud provider should be added. For each platform with `external` configuration, the `cloud-config` flag should not be set.

### Kube-api-server

No changes for the current configuration are needed

### Kube-controller-manager

TBD

#### Upgrade/Downgrade strategy

Upstream [proposal](https://github.com/kubernetes/enhancements/blob/473d6a094f07e399079d1ce4698d6e52ddcc1567/keps/sig-cloud-provider/20190422-cloud-controller-manager-migration.md#motivation) describes main pinpoints for migration between in-tree and out-of-tree for HA clusters. They propose implementation for cross-namespace leadership delegation, which would help Openshift to preserve HA during upgrages.

In case the cluster does not require to be HA, then upgrading under CVO management, which will lead to two simultanious replicas of `cloud-controllers` running at the same time at some moment is not a problem. Each provider, therefore supports multiple options to run the `cloud-controller`:

1) As a `DaemonSet`. This is the HA option, where the leader-election on the running pod ensuring a single replica is serving cluster at any point in time.
2) As a `Pod`. For Openshift this option is not sufficient, so non-HA environment will run this pod under `Deployment` or as a static resource managed from the `Node` filesystem in the `openshift-cloud-controller` namespace.

#### Examples

##### AWS

Main repository: https://github.com/kubernetes/cloud-provider-aws/
Sample manifests: https://github.com/kubernetes/cloud-provider-aws/tree/master/manifests

To try external cloud-controller-manager, siply run:
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

### Test Plan

- Each provider at the time of migration should have a working CI implemetation, which will assist in testing the provisioning with out-of-tree support enabled.
- Ensure that transition between in-tree and out-of-tree will be handled by the `Infrastracture` `external-cloud-config` field being set.
- Ensure in-tree and out-of-tree providers could co-exist in the cluster if this is supported.
- Ensure that upgrade from a release running on in-tree provider only to the out-of-tree succeeds.

##### Removing a deprecated feature

Upstream is currently planning to remove support for in-tree providers with OCP-4.8 (kubernetes 1.21) - based on the slack [conversation](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600). The enchancement [describes](https://github.com/kubernetes/enhancements/blob/473d6a094f07e399079d1ce4698d6e52ddcc1567/keps/sig-cloud-provider/20190125-removing-in-tree-providers.md) steps towards this process. Our goal is to have a working alternative for in-tree providers ready to be used on-par with old implementation befor the upstream release will remove this feature from main `k/k` repository.

## Timeline

Follow upstream discussions and implementation/releases

## Kubernetes 1.18

- vSphere support graduaded to beta: https://github.com/kubernetes/enhancements/issues/670

## Kubernetes 1.19

- Azure support goes into beta: https://github.com/kubernetes/enhancements/issues/667

## Infrastructure Needed

(Possibly) Forks for out-of-tree provider repositories:
- [AWS](https://github.com/kubernetes/cloud-provider-aws)
- [GCP](https://github.com/kubernetes/cloud-provider-gcp)
- [Azure](https://github.com/kubernetes-sigs/cloud-provider-azure)
- [Vsphere](https://github.com/kubernetes-sigs/cloud-provider-vsphere)

