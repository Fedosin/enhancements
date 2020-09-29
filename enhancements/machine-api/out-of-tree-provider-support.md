---
title: out-of-tree-provider-support
authors:
  - "@Danil-Grigorev"
  - "@Fedosin"
reviewers:
  - "@enxebre"
  - "@JoelSpeed"
  - "@crawford"
  - "@derekwaynecarr"
  - "@eparis"
  - "@mrunalp"
  - "@sttts"
approvers:
  - "@enxebre"
  - "@JoelSpeed"
  - "@crawford"
  - "@derekwaynecarr"
  - "@enxebre"
  - "@eparis"
  - "@mrunalp"
  - "@sttts"
creation-date: 2020-08-31
last-updated: 2020-09-29
status: implementable
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

This enhancement proposal describes the migration of cloud platforms from the deprecated [in-tree cloud providers](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#openstack) to [Cloud Controller Manager](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-openstack-cloud-controller-manager.md#get-started-with-external-openstack-cloud-controller-manager-in-kubernetes) services that implement `external cloud provider` [interface](https://github.com/kubernetes/cloud-provider).

As a pioneer platform, it is proposed to use OpenStack.

## Motivation

Using Cloud Controller Managers (CCMs) is the Kubernetes' [preferred way](https://kubernetes.io/blog/2019/04/17/the-future-of-cloud-providers-in-kubernetes/) to interact with underlying cloud platforms as it provides more flexibility and freedom for developers. It replaces existing in-tree cloud providers, which have been deprecated and will be permanently removed approximately in Kubernetes [v1.21](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600). But they are still used in OpenShift and we must start a smooth migration towards CCMs.

Another motivation is to be closer to upstream by helping developing of Cloud Controller Managers for various platforms, which is benefiting both OpenShift and Kubernetes.

The change will help adding support for other cloud platforms, such as [Digital Ocean](https://github.com/digitalocean/digitalocean-cloud-controller-manager) or [Alibaba Cloud](https://github.com/kubernetes/cloud-provider-alibaba-cloud).

It's especially important to do this for OpenStack because switching to the external cloud provider fixes many issues and limitations with the in-tree cloud provider, such as it's reliance on [Nova metadata service](https://docs.openstack.org/nova/latest/admin/metadata-service.html). For OpenStack platform, this means the possibility for deploying on provider networks and at the edge.

### Goals

- Prepare OpenShift to start using CCM instead of deprecated in-tree cloud providers.
- Provide the means to deploy and manage various CCMs. It is proposed to create a new cluster operator - `cluster-cloud-controller-manager-operator` for this.
- Define and implement an upgrading path from older OpenShift versions with the in-tree cloud provider to CCM'ed ones.
- Define and implement a downgrading path from newer OpenShift versions with the CCM to those that use the in-tree cloud provider.

### Non-Goals

- [CSI driver migration](https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/) is out of scope of this work.

## Proposal

Our main goal is to start using Cloud Controller Manager in OpenShift 4, and make a seamless transition for all currently supported providers that require `cloud provider` interface: OpenStack, AWS, Azure, GCP and vSphere.

To maintain the lifecycle of the CCM we want to implement a cluster operator called `cluster-cloud-controller-manager-operator`, that will handle all administrative tasks: deploy, restore, upgrade, and so on.

### Implementation Details

The `cluster-cloud-controller-manager-operator` implementation will be hosted within OpenShift repository in https://github.com/openshift/cluster-cloud-controller-manager-operator. That repository will manage CCM provisioning for mentioned cloud platforms.

- All platform CCMs will be provisioned in `openshift-cloud-controller-manager` namespace.
- The Operator will be deployed in `openshift-cloud-controller-manager-operator` namespace.

#### Kubelet

OpenShift manages `kubelet` configuration with `machine-config-operator`. The actual `cloud-config` arguments are set externally, based on the `Infrastructure` resource values on https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357, where the support for `external` cloud provider will be added. For each platform with `external` configuration, the `cloud-config` flag will not be set.

Kubelet is tightly coupled with `CCMs`, due to its [usage](https://github.com/kubernetes/kubernetes/blame/323f34858de18b862d43c40b2cced65ad8e24052/pkg/kubelet/cloudresource/cloud_request_manager.go#L98-L102) of cloud-provider interfaces, such as `Instances` and `NodeAdresses`. Whenever there is a need to bootstrap any `Node`, `kubelet` will collect missing information about the instance IP addresses, type and zone. This information could later be used by `kube-scheduler`.

During this procedure all new `Nodes` in the cluster are [tainted](https://github.com/kubernetes/kubernetes/blob/c53ce48d48372c30053a9b67c6d1dee237b5cd69/pkg/kubelet/kubelet_node_status.go#L307-L311) with `node.cloudprovider.kubernetes.io/uninitialized: true`. This taint is removed after the [initialization](https://github.com/kubernetes/kubernetes/blob/46d522d0c8e94edb6d29606a28785e35259badd7/staging/src/k8s.io/cloud-provider/controllers/node/node_controller.go#L374-L376) by CCM.

#### Kube Controller Manager and Kube API Server

Configuration of these components is managed by `cluster-kube-controller-manager-operator` and `cluster-kube-apiserver-operator` respectively. Both operators use [`configobserver`](https://github.com/openshift/library-go/blob/16d6830d0b80dc2a3315207116d009ed2dd4cebf/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go#L140) module from `library-go` where we set what cloud provider should be used for the given platform. When a platform is being switched to CCM, we shouldn't set `cloudProvider` value there and leave it empty.

Next step is to bump `library-go` for the operators.

#### Bootstrap changes

For the initial transition from in-tree to the out-of-tree, `CCM` pod will be created as static pod on the bootstrap node, to ensure swift removal of the `node.cloudprovider.kubernetes.io/uninitialized: true` taint from a newly created `Nodes`. Later stages, including cluster upgrades will be managed by an operator, which will ensure stability of the configuration, and will run a `CCM` in a `Deployment`.

#### Operator resource management strategy

Every cloud manages it's configuration differently, but share a common overall structure.

1. `cloud-controller-manager` and related resources are deployed under a single namespace.
2. Each pod (`CCM`, `cloud-node-manager`, etc.) will be managed by a `Deployment`.
3. The pods are expected to run on the control-plane `Nodes` using `hostNetwork`.

Operator will manage:

- Static infrastructure resources creation, such as namespaces or service accounts will be delegated to the [CVO](https://github.com/openshift/cluster-version-operator).
- Operator will populate, create and manage the cloud-specific set of `Deployments`, based on the selected `platform` in the `Infrastructure` resource.
- The `--cloud-provider` and the `--cloud-config` arguments will be populated using existing `library-go` based implementation from the [cluster-kube-controller-manager-operator](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85) to share common approach with the bootstrap specific static-pod configuration.
- Operator will be build with a goal to achieve provider decoupling and potential switching between providers in the cluster.

## Upgrade/Downgrade strategy

A set of `cloud-controller-manager` pods will be running under a `Deployment` resource, provisioned by the operator. The leader-election will preserve the leadership during the cluster updates over cloud-specific loops.

Upgrade from previous versions of OpenShift on OpenStack will look like:

- On the initial stage of upgrading `cluster-version-operator` starts `cluster-cloud-controller-manager-operator`.

- `cluster-cloud-controller-manager-operator` creates a static pod manifest for CCM.

- `cluster-cloud-controller-manager-operator` creates all required resources for CCM (Namespace, RBAC, Service Account).

- `kube-apiserver` and `kube-controller-manager` are restarted without the `--cloud-provider` option.

- `kubelet` is restarted with `--cloud-provider external` option.

Downgrade:

- An old machine configuration is applied, which causes `kubelet` to restart with `--cloud-provider openstack` option.

- `kube-apiserver` and `kube-controller-manager` are restarted with the `--cloud-provider openstack` option.

- `cluster-cloud-controller-manager-operator` stops working.

### Version Skew Strategy

See the upgrade/downgrade strategy.

### Action plan (OpenStack)

#### Build OpenStack CCM image by OpenShift automation

To start using OpenStack CCM in OpenShift we need to build its image and make sure it is a part of the OpenShift release image. The CCM image should be automatically tested before it becomes available.
The upstream repo provides the Dockerfile, so we can reuse it to complete the task. Upstream image is already available in [Dockerhub](https://hub.docker.com/r/k8scloudprovider/openstack-cloud-controller-manager).

Actions:

- Configure CI operator to build OpenStack CCM image.

CI operator will run containerized and End-to-End tests and also push the resulting image in the OpenShift Quay account.

#### Test Cloud Controller Manager manually

When all required components are built, we can manually deploy OpenStack CCM and test how it works.

Actions:

- Manually install CCM's [daemonset](https://github.com/kubernetes/cloud-provider-openstack/blob/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml) on a working OpenShift cluster deployed on OpenStack.

- Update configuration of `kubelet` by replacing `--cloud-provider openstack` with `--cloud-provider external` and removing `--cloud-config` parameters.
For `kube-apiserver` and `kube-controller-manager` we need to remove both `--cloud-provider` and `--cloud-config` parameters and restart `kubelet`.

**Note:** Example of a manual testing: https://asciinema.org/a/303399?speed=2

#### Write Cluster Cloud Controller Manager Operator

The operator should be able to create, configure and manage different CCMs (including one for OpenStack) in OpenShift. The architecture of the operator will be following practices from [Kubernetes Controller Manager](https://github.com/openshift/cluster-kube-controller-manager-operator) and [Kubernetes API server](https://github.com/openshift/cluster-kube-apiserver-operator) operators.

Actions:

- Create a new repo in OpenShift’s github: https://github.com/openshift/cluster-cloud-controller-manager-operator (Done)

- Implement the operator, using [library-go](https://github.com/openshift/library-go) primitives.

#### Build the operator image by OpenShift automation

To start using CCM operator in OpenShift we need to build its image and make sure it is a part of the OpenShift release image. The image should be automatically tested before it becomes available. Dockerfile should be a part of the operator's repo.

Actions:

- Configure CI operator to build CCM operator image.

CI operator will run containerized and End-to-End tests and also push the resulting image in the OpenShift Quay account.

#### Integrate the solution with OpenShift

Actions:

- Make sure Cinder CSI driver is supported in OpenShift.

- Add CCM operator support in Cluster Version Operator. CCM operator will be deployed on early stages of installation and then it deploys and configures OpenStack CCM itself.

- Change the config observer in [library-go](https://github.com/openshift/library-go/blob/16d6830d0b80dc2a3315207116d009ed2dd4cebf/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go#L154) to disable setting `--cloud-provider` and `--cloud-config` parameters for OpenStack. Then the library should be bumped in `cluster-kube-apiserver-operator` and `cluster-kube-controller-manager-operator` respectively.

- Change `kubelet` configuration for OpenStack in the [MCO](https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357) to adopt external cloud provider.

### Cloud Controller Manager installation workflow

Starting from OpenShift 4.7 the installation of OpenShift on OpenStack will be next:

- All Kubernetes components that require cloud provider functionality (`kubelet`, `kube-apiserver`, `kube-controller-manager`) are preliminary configured to use `external` cloud provider. In other words, `kubelet` is launched with the `--cloud-provider external` only; `kube-apiserver`and `kube-controller-manager` specify neither `--cloud-provider` nor `--cloud-config` parameters.

- CCM operator provides initial manifests that allow to deploy CCM on the bootstrap machine with `bootkube.sh` [script](https://github.com/openshift/installer/blob/master/data/data/bootstrap/files/usr/local/bin/bootkube.sh.template).

- `cluster-version-operator` starts `cluster-cloud-controller-manager-operator`.

- `cluster-cloud-controller-manager-operator` checks if it runs on OpenStack, populates [configuration](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-openstack-cloud-controller-manager.md#config-openstack-cloud-controller-manager) for OpenStack Cloud Controller Manager, creates a static [OpenStack CCM pod](https://github.com/kubernetes/cloud-provider-openstack/blob/master/manifests/controller-manager/openstack-cloud-controller-manager-pod.yaml) and monitors its status.

#### Configuration of Cloud Controller Manager

At the initial stage of CCM installation installer creates a config map with `cloud.conf` key that contains configuration of CCM in `ini` format. The contents of `cloud.conf` are static and generated by the installer:

```txt
[Global]
secret-name = openstack-credentials
secret-namespace = kube-system
```

Real config is also generated by the installer and available in the given secret. Based on this static config, CCM fetches the real one from the secret and uses it.

**NOTE**: This is also how the in-tree cloud provider for OpenStack is configured in this moment. This part is already implemented and available in [the installer](https://github.com/openshift/installer/blob/master/pkg/asset/manifests/openstack/cloudproviderconfig.go).

#### Examples for other cloud-controller-manager configurations

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

vSphere introduced an additional service to expose internal API server: https://github.com/kubernetes/cloud-provider-vsphere/blob/master/manifests/controller-manager/vsphere-cloud-controller-manager-ds.yaml#L64

### Risks and Mitigations

#### Specific CCM doesn’t work properly on OpenShift

The `CCMs` have not been tested on either OSP or OCP. Unknown issues may arise.

Severity: medium-high
Likelihood: high

#### OpenShift components can't work with the CCM properly

Since OpenShift imposes additional limitations compared to Kubernetes, some collisions are possible.
So far there are no CCMs available in OpenShift, and there can be some problems with operability.

Severity: medium-high
Likelihood: low

#### CSI driver is not available in OpenShift

CCM doesn't support in-tree `PV` manager implementation. A CSI driver now becomes a mandatory requirement for every out-of-tree provider. Additionally the CSI should support data migration between the in-tree storage solution and the plugin provided storage interfaces.

Severity: medium-high
Likelihood: low

### Test Plan

- Make sure the `CCM` implementation for "any" cloud is up and running in a bootstrap phase before or shortly after the `kubelet` is running on any `Node`.
- Ensure that transition between the in-tree and the out-of-tree will be handled by the established architecture with the new relase payload, where all of the existing `Nodes` are operational during and after the upgrade time, and the bootstrap procedure for new ones is successfull.
- Ensure that upgrade from a release running on the in-tree provider only to the out-of-tree succeeds, and the downgrade is also supported.
- Each provider should have a working CI implemetation, which will assist in testing the provisioning and upgrades with out-of-tree support enabled.

## Open questions

1. Should we reuse the existing cloud provider config or generate a new one?
CCM config is backward compatible with the in-tree cloud provider. It means we can reuse it.

2. (Openstack) How to migrate PVs created by the in-tree cloud provider?
[CSIMigration](https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/) looks like the best option, especially if it is GA in 1.19.

## FAQ

Q: Can we disaster-recover a cluster without CCM running?
A: We can't. Basically, without CCM running, nodes can only join the cluster, but they will be unschedulable. Which means it's impossible to start any workloads on the nodes. [Source](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/#running-cloud-controller-manager)

Q: What is non-functional in a cluster during bootstrapping until CCM is up?
A: If CCM is not available, new nodes in the cluster will be left unschedulable.

Q: Who talks to CCM?
A: From architectural point of view, only Kube API server is able to communicate with CCM. [Source](https://kubernetes.io/docs/concepts/architecture/cloud-controller/#design). All requests to CCM are sent by `kubelet`. KCM doesn't interract with CCM.

Q: Does CCM provide an API? How is it hosted? HTTPS endpoint? Through load balancer?
A: CCM provides a gRPC interface on port [10258](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cloud-provider/ports.go#L22). Like KCM, CCM supports high available setup using leader election (which is on by default). [Source](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/#requirements).

Q: What are the thoughts about certificate management?
A: Certificate management should be performed by the CCM operator, like it's done in the KCM operator.

Q: What happens if the KCM leader has no access to CCM? Can it continue its work? Will it give up leadership?
A: Since KCM does not communicate with CCM it can continue to work if CCM is not available.

Q: Does every node need a CCM?
A: No. Control plane nodes only. [Source](https://kubernetes.io/docs/concepts/overview/components/#cloud-controller-manager)

Q: How does SDN depend on CCM?
A: Most likely they are not related.

## Other notes

##### Removing a deprecated feature

Upstream is currently planning to remove support for the in-tree providers with OCP-4.8 (kubernetes 1.21) - based on the slack [conversation](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600). The enchancement [describes](https://github.com/kubernetes/enhancements/blob/473d6a094f07e399079d1ce4698d6e52ddcc1567/keps/sig-cloud-provider/20190125-removing-in-tree-providers.md) steps towards this process. Our goal is to have a working alternative for the in-tree providers ready to be used on-par with old implementation befor the upstream release will remove this feature from core Kubernetes repository.

## Timeline

### Upstream

Follow the Kubernetes community discussions and implementation/releases

- vSphere support has graduated to beta in [v1.18](https://github.com/kubernetes/enhancements/issues/670)

- Azure support goes into beta in [v1.20](https://github.com/kubernetes/enhancements/issues/667)

### Openshift

- Migrate OpenStack platform on CCM in v4.7.

- Migrate vSphere, AWS, Azure and GCP platforms on CCM in 4.8.

## Infrastructure Needed

Additional infrastructure for OpenStack and vSphere may be required to test how CCM works with self-signed certificates. Current CI doesn't allow this.

Other platforms do not require additional infrastructure.

### New repositories

We need to forks next repositories with CCM implementations:

- [AWS](https://github.com/kubernetes/cloud-provider-aws)
- [GCP](https://github.com/kubernetes/cloud-provider-gcp)
- [Azure](https://github.com/kubernetes-sigs/cloud-provider-azure)
- [vSphere](https://github.com/kubernetes/cloud-provider-vsphere)

OpenStack repo is already [cloned](https://github.com/openshift/cloud-provider-openstack) and the CCM image is shipped

Mandatory operator repository:

- [CCM Operator](https://github.com/openshift/cluster-cloud-controller-manager-operator)

Each CCM will be built with an image of its own, which will be included in the OpenShift payload.

## Additional Links

- [Kubernetes Cloud Controller Managers](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)

- [Cloud Controller Manager Administration](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)

- [The Kubernetes Cloud Controller Manager](https://medium.com/@m.json/the-kubernetes-cloud-controller-manager-d440af0d2be5) article
