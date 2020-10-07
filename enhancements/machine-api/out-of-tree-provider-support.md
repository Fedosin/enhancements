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
last-updated: 2020-10-07
status: implementable
replaces:
superseded-by:
---

# Add support for out-of-tree cloud provider integration

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement proposal describes the migration of cloud platforms from the deprecated [in-tree cloud providers](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#openstack) to [Cloud Controller Manager](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-openstack-cloud-controller-manager.md#get-started-with-external-openstack-cloud-controller-manager-in-kubernetes) services that implement `external cloud provider` [interface](https://github.com/kubernetes/cloud-provider).

As a pioneer platform, it is proposed to use OpenStack.

## Motivation

Using Cloud Controller Managers (CCMs) is the Kubernetes' [preferred way](https://kubernetes.io/blog/2019/04/17/the-future-of-cloud-providers-in-kubernetes/) to interact with underlying cloud platforms as it provides more flexibility and freedom for developers. It replaces existing in-tree cloud providers, which have been deprecated and will be permanently removed approximately in Kubernetes [v1.21](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600). But they are still used in OpenShift and we must start a smooth migration towards CCMs.

Another motivation is to be closer to upstream by helping develop Cloud Controller Managers for various platforms, which is benefiting both OpenShift and Kubernetes.

The change will help adding support for other cloud platforms, such as [Packet](https://github.com/packethost/packet-ccm), [Digital Ocean](https://github.com/digitalocean/digitalocean-cloud-controller-manager) or [Alibaba Cloud](https://github.com/kubernetes/cloud-provider-alibaba-cloud).

It's especially important to do this for OpenStack because switching to the external cloud provider fixes many issues and limitations with the in-tree cloud provider, such as it's reliance on [Nova metadata service](https://docs.openstack.org/nova/latest/admin/metadata-service.html). For OpenStack platform, this means the possibility for deploying on provider networks and at the edge.

### Goals

- Prepare OpenShift components to accommodate the CCMs instead of deprecated in-tree cloud providers.
- Provide the means to allow management of generic `CCM` component based on the platform specific upstream implementation.
- Define and implement an upgrading and downgrading paths between the in-tree and the out-of-tree cloud provider configurations.
- Ensure the transition will ensure the feature parity between the in-tree and the out-of-tree provider.
- Consider support for multiple providers within a single cluster as a possible future configuration (and make sure there are no clashes between provider configurations).

### Non-Goals

- Force an immediate transition for all cloud providers from in-tree support to their out-of-tree counterparts.
- [CSI driver migration](https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/) is out of scope of this work.

## Proposal

Our main goal is to start using Cloud Controller Manager in OpenShift 4, and make a seamless transition for all currently supported providers that require `cloud provider` interface: OpenStack, AWS, Azure, GCP and vSphere.

To maintain the lifecycle of the CCM we want to implement a cluster operator called `cluster-cloud-controller-manager-operator`, that will handle all administrative tasks: deploy, restore, upgrade, and so on.

### User Stories

#### Story 1

As a cloud developer, I’d like to improve support for cloud features for OpenShift. I'd like to develop and test new cloud providers which do not ship with default Kubernetes distribution, and require support for external cloud controller manager.

#### Story 2

As a developer I'd like to implement, build and release fixes for cloud provider controllers independently from the Kubernetes core, and assume they will first land upstream and then will be carried over into OpenShift distribution with less effort.

#### Story 3

As an OpenShift developer responsible for the cloud controllers, I want to share more code with upstream Kubernetes in order to ease feature and bug fix contributions both ways.

#### Story 4

We’d like to discuss technical details related to a specific cloud in a SIG meeting with people who are also involved into development in this domain, and that way gain useful insights into the cloud infrastructure, improve the overall quality of our features, stay on top of the new features, and improve the relations with maintainers outside of our company, which nevertheless share with us a common goal.

### Implementation Details

The `cluster-cloud-controller-manager-operator` implementation will be hosted within its own OpenShift [repository](https://github.com/openshift/cluster-cloud-controller-manager-operator). The operator will manage `CCM` provisioning for supported cloud providers.

- All platform CCMs will be provisioned in `openshift-cloud-controller-manager` namespace.
- The Operator will be deployed in `openshift-cloud-controller-manager-operator` namespace.

#### Resource changes

#### Kubelet

OpenShift manages `kubelet` configuration with `machine-config-operator`. The actual `cloud-config` arguments are set externally, based on the `Infrastructure` [resource values](https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357), where the support for `external` cloud providers will be added. For each platform with `external` configuration, the `cloud-config` flag will not be set.

Kubelet is tightly coupled with `CCMs`, due to its [usage](https://github.com/kubernetes/kubernetes/blame/323f34858de18b862d43c40b2cced65ad8e24052/pkg/kubelet/cloudresource/cloud_request_manager.go#L98-L102) of cloud-provider interfaces, such as `Instances` and `NodeAdresses`. Whenever there is a need to bootstrap any `Node`, `kubelet` will collect missing information about the instance IP addresses, type and zone. This information could later be used by `kube-scheduler`.

During this procedure all new `Nodes` in the cluster are [tainted](https://github.com/kubernetes/kubernetes/blob/c53ce48d48372c30053a9b67c6d1dee237b5cd69/pkg/kubelet/kubelet_node_status.go#L307-L311) with `node.cloudprovider.kubernetes.io/uninitialized: true`. This taint is removed after the [initialization](https://github.com/kubernetes/kubernetes/blob/46d522d0c8e94edb6d29606a28785e35259badd7/staging/src/k8s.io/cloud-provider/controllers/node/node_controller.go#L374-L376) by `CCM`.

#### Kube Controller Manager and Kube API Server

Configuration of these components is managed by `cluster-kube-controller-manager-operator` and `cluster-kube-apiserver-operator` respectively. Both operators use [`configobserver`](https://github.com/openshift/library-go/blob/16d6830d0b80dc2a3315207116d009ed2dd4cebf/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go#L140) module from `library-go` where we set what cloud provider should be used for the given platform. When a platform is being switched to `CCM`, we [must not](https://meet.google.com/linkredirect?authuser=0&dest=https%3A%2F%2Fkubernetes.io%2Fdocs%2Ftasks%2Fadminister-cluster%2Frunning-cloud-controller%2F%23running-cloud-controller-manager) set `cloudProvider` value there and leave it empty.

Next step is to bump `library-go` for the operators.

##### Pre-release / Development

In order to achieve desired cluster configuration for cloud components during development, we need to add an annotation on the `Infrastructure` [resource](https://github.com/openshift/api/blob/c2f7aea5d89ee87d7510b50bdb08dde9028a53ef/config/v1/types_infrastructure.go#L11).

Proposed annotation: `infrastructure.openshift.io/external-cloud-provider`.

The annotation will determine the architectural change in the cluster configuration. If this annotation would be found, other components, such as `kube-apiserver`, `kube-controller-manager` and `kubelet` will set their flags accordingly, to allow hosting external `CCM` for any platform.

The annotation will be a temporary measure to simplify the operator development allowing multiple components to merge implementations of this feature independently before switching on the feature by changing the value of the annotation. Once all providers are moved out-of-tree, the annotation will be safe to remove.

To ensure a smooth transition from the `in-tree` selection, `library-go` [implementation](https://github.com/openshift/library-go/blob/master/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go) will include an additional argument for the `cloudProviderObserver` structure, which will allow to preserve initial configuration strategy, until the new `observeExternal` flag will be set to `false`:

```go
type cloudProviderObserver struct {
    targetNamespaceName     string
    cloudProviderNamePath   []string
    cloudProviderConfigPath []string
    observeExternal         bool
}
```

The `observeExternal` field will determine, when the annotation is present on the `Infrastructure` resource, it will be taken into account while establishing needed flags for a Kubernetes binaries. It will make the observer type to the `cloud-config` file, and set flags as was described in the `kubelet`, `kube-apiserver` and `kube-controller-manager` sections.

#### Bootstrap changes

For the initial transition from in-tree to the out-of-tree, `CCM` pod will be created as static pod on the bootstrap node, to ensure swift removal of the `node.cloudprovider.kubernetes.io/uninitialized: true` taint from any newly created `Nodes`. Later stages, including cluster upgrades will be managed by an operator, which will ensure stability of the configuration, and will run a `CCM` in a `Deployment`. Initial usage of the static pod is justified by the need to run `CCM` pod before the `kube-apiserver`, `etcd` and the `kube-scheduler` will get healthy, so the functionality does not rely on them.

`CCM` operator will provide initial manifests that allow to deploy `CCM` on the bootstrap machine with `bootkube.sh` [script](https://github.com/openshift/installer/blob/master/data/data/bootstrap/files/usr/local/bin/bootkube.sh.template).

At the initial stage of `CCM` installation installer creates a config map with `cloud.conf` key that contains configuration of `CCM` in `ini` format. The contents of `cloud.conf` are static and generated by the installer:

Example for OpenStack:

```txt
[Global]
secret-name = openstack-credentials
secret-namespace = kube-system
```

Resources deployed here will be destroyed with the bootstrap stage cleanup.

#### Operator resource management strategy

Every cloud manages it's configuration differently, but shares a common overall structure.

1. `cloud-controller-manager` and related resources are deployed under a single namespace.
2. Each pod (`CCM`, `cloud-node-manager`, etc.) will be managed by a `Deployment`.
3. The pods are expected to run on the control-plane `Nodes` using `hostNetwork`.

Operator will manage:

- Static infrastructure resources creation, such as namespaces or service accounts will be delegated to the [CVO](https://github.com/openshift/cluster-version-operator).
- Operator will populate, create and manage the cloud-specific set of `Deployments`, based on the selected `platform` in the `Infrastructure` resource.
- `Deployments`, provisioned by the operator will run the pods on the `control-plane` nodes, and will tolerate `cloudprovider.kubernetes.io/uninitialized` and `node-role.kubernetes.io/master` taints.
- The `--cloud-provider` and the `--cloud-config` arguments will be populated using existing `library-go` based implementation from the [cluster-kube-controller-manager-operator](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85) to share a common approach with the bootstrap specific static-pod configuration.
- Operator will own a `cluster-operator` resource, which will report the readiness of all workloads at post-install phase. The conditions reported will help other dependent components, such as [openshift-ingress](https://github.com/openshift/cluster-ingress-operator).
- Operator will be built with a goal to achieve provider decoupling and potential switching between providers in the cluster, and will consider support to run multiple providers in a cluster simultaneously (long-term goal).

#### CVO managemnet

The operator image will be build and included in every release payload, and expected to be deployed and running at kuberentes operators level ([10-29](https://github.com/openshift/cluster-version-operator/blob/34010292c3abd2582b687e9c0ef76c5924998f39/docs/dev/operators.md#how-do-i-get-added-as-a-special-run-level)) right after the `kube-controller-manager-operator` at runlevel [25](https://github.com/openshift/cluster-kube-controller-manager-operator/tree/master/manifests).

Proposed `CCMO` manifests runlevel is `26`.

#### Cloud provider management

Each cloud provider will be hosted under its own repository. Those repositories will be initially bootstrapped by forking upstream ones.

The `CVO` will be responsible for provisioning static resources, such as `SA`, `RBAC`, `ServiceMonitors` for metrics for the operator. Then, the operator will  be responsible for constructing and creating cloud provider resources.

Forked provider repositories will also contain cloud-provider specific code, and a set of binaries for each of them to build and run. Each provider will be build inside a single image, which will be included in the release payload. The responsibility for the operator will be to choose which provider image should be used in a cluster at any moment.

#### CCM migration from KCM pod

Currently the cloud-specific controller loops are running inside the `kcm` pod. We need to preserve leadership over these parts during migration process. This includes migration from the `openshift-kube-controller-manager` namespace to the `openshift-cloud-controller-manager` namespace, which is not possible with current `leader-election` implementation.

Upstream [proposal](https://github.com/kubernetes/enhancements/blob/473d6a094f07e399079d1ce4698d6e52ddcc1567/keps/sig-cloud-provider/20190422-cloud-controller-manager-migration.md#motivation) describes the main implementation details for `Lease` resource. This cluster-scoped resource maintains cross-namespaced component leadership, and would help OpenShift to preserve HA during migration from the in-tree to the out-of-tree.

#### Credentials management

The operator is expected to be integrated with the [cloud-credentials-operator](https://github.com/openshift/cloud-credential-operator), and issue the fine grained credentials for the cloud components.

*Bootstrap phase:*

- Initial configuration with static pods is expected to be using `ConfigMap` credentials, similarly to what is [done](https://github.com/openshift/installer/blob/ba6d7fe087eada5a4fc260064c85a484a8c45aaf/data/data/bootstrap/files/usr/local/bin/bootkube.sh.template#L162) in `kcm` operator.
- Once it is supported, the static pods will also use the `CredentialsRequest` to get cloud credentials.

*Post-install phase:*

- Using `CredentialsRequest` is the default option, every supported `CCM` within the `Deployment` will request its own set of credentials.
- A set of `CredentialsRequest` resources will be hosted under each provider repository, and created by the `CVO`, similarly to the [machine-api](https://github.com/openshift/machine-api-operator/blob/6f629682b791a6f4992b78218bfc6e41a32abbe9/install/0000_30_machine-api-operator_00_credentials-request.yaml) approach.

### Upgrade/Downgrade strategy

A set of `cloud-controller-manager` pods will be running under a `Deployment` resource, provisioned by the operator. The leader-election will preserve the leadership during the cluster updates over cloud-specific loops.

Upgrade from previous versions of OpenShift on OpenStack will look like:

#### 4.6 -> 4.7

Upgrade (all platforms):

- The `cluster-version-operator` starts `cluster-cloud-controller-manager-operator`.

Downgrade (all platforms):

- `cluster-cloud-controller-manager-operator` de-provisions CCM `deployments` and stops working.

The steps below apply to OpenStack only:

Upgrade:

- The `cluster-version-operator` starts `cluster-cloud-controller-manager-operator`.
- `cluster-cloud-controller-manager-operator` creates all required resources for CCM (RBAC, Service Account, etc.).
- `kube-apiserver` and `kube-controller-manager` are restarted without the `--cloud-provider` option.
- `kubelet` is restarted with `--cloud-provider=external` option.

Downgrade:

- `cluster-cloud-controller-manager-operator` de-provisions CCM `deployments` and stops working.
- `kube-apiserver` and `kube-controller-manager` are restarted with the `--cloud-provider=<some-platform>` option.
- An old machine configuration is applied, which causes `kubelet` to restart with `--cloud-provider=<some-platform>` option.

#### 4.7 -> 4.8

Upgrade:

- `cluster-cloud-controller-manager-operator` creates resources for any in-tree provider defaulting to this in 4.8 (expecting: AWS, Azure, GCP, vSphere).
- `kube-apiserver` and `kube-controller-manager` are restarted without the `--cloud-provider` option.
- `kubelet` is restarted with `--cloud-provider=external` option.

Downgrade:

- `cluster-cloud-controller-manager-operator` de-provisions CCM resources for any platform, except OpenStack.
- `kube-apiserver` and `kube-controller-manager` are restarted with the `--cloud-provider=<some-platform>` option.
- An old machine configuration is applied, which causes `kubelet` to restart with `--cloud-provider=<some-platform>` option (excluding OpenStack).

#### 4.8 and later

Upgrade:

- `cluster-cloud-controller-manager-operator` updates the state of resources for any in-tree provider defaulting to this since previous releases.

Downgrade:

- `cluster-cloud-controller-manager-operator` downgrades the state of resources for any in-tree provider defaulting to this since previous releases.

### Version Skew Strategy

See the upgrade/downgrade strategy.

### Action plan (for OpenStack)

#### Build OpenStack CCM image by OpenShift automation

To start using OpenStack CCM in OpenShift we need to build its image and make sure it is a part of the OpenShift release image. The CCM image should be automatically tested before it becomes available.
The upstream repo provides the Dockerfile, so we can reuse it to complete the task. Upstream image is already available in [Dockerhub](https://hub.docker.com/r/k8scloudprovider/openstack-cloud-controller-manager).

Actions:

- Configure CI operator to build OpenStack CCM image.

CI operator will run containerized and End-to-End tests and also push the resulting image in the OpenShift Quay account.

#### Test Cloud Controller Manager manually

When all required components are built, we can manually deploy OpenStack CCM and test how it works.

Actions:

- Manually install CCM's example [daemonset](https://github.com/kubernetes/cloud-provider-openstack/blob/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml) on a working OpenShift cluster deployed on OpenStack.

- Update configuration of `kubelet` by replacing `--cloud-provider openstack` with `--cloud-provider external` and removing `--cloud-config` parameters.
For `kube-apiserver` and `kube-controller-manager` we need to remove both `--cloud-provider` and `--cloud-config` parameters and restart `kubelet`.

- Check the CCM functionality is operational at this point.

**Note:** Example of a manual testing: https://asciinema.org/a/303399?speed=2

#### Write Cluster Cloud Controller Manager Operator

The operator should be able to create, configure and manage different CCMs (including one for OpenStack) in OpenShift. The architecture of the operator will be following practices from [Kubernetes Controller Manager](https://github.com/openshift/cluster-kube-controller-manager-operator) and [Kubernetes API server](https://github.com/openshift/cluster-kube-apiserver-operator) operators.

Actions:

- Create a new repo in OpenShift’s github: https://github.com/openshift/cluster-cloud-controller-manager-operator (Done)

- Implement the operator, using [library-go](https://github.com/openshift/library-go) primitives.

#### Build the operator image by OpenShift automation

To start using the CCM operator in OpenShift we need to build its image and make sure it is a part of the OpenShift release image. The image should be automatically tested before it becomes available. Dockerfile should be a part of the operator's repo.

Actions:

- Configure CI operator to build CCM operator image.

CI operator will run containerized and End-to-End tests and also push the resulting image in the OpenShift Quay account.

#### Integrate the solution with OpenShift

Actions:

- Make sure the Cinder CSI driver is supported in OpenShift.

- Add CCM operator support in Cluster Version Operator. CCM operator will be deployed on early stages of installation and then it deploys and configures OpenStack CCM itself.

- Change the config observer in [library-go](https://github.com/openshift/library-go/blob/16d6830d0b80dc2a3315207116d009ed2dd4cebf/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go#L154) to disable setting `--cloud-provider` and `--cloud-config` parameters for OpenStack. Then the library should be bumped in `cluster-kube-apiserver-operator` and `cluster-kube-controller-manager-operator` respectively.

- Change `kubelet` configuration for OpenStack in the [MCO](https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357) to adopt external cloud providers.

#### Examples for other cloud-controller-manager configurations

- AWS: https://github.com/kubernetes/cloud-provider-aws/
- Azure: https://github.com/kubernetes-sigs/cloud-provider-azure
Azure separated the `CCM` and the "cloud-node-manager" (`CNM`) into two binaries: https://github.com/kubernetes-sigs/cloud-provider-azure/tree/master/cmd.
- GCP: https://github.com/kubernetes/cloud-provider-gcp
- vSphere: https://github.com/kubernetes/cloud-provider-vsphere
vSphere introduced an additional service to expose internal API server: https://github.com/kubernetes/cloud-provider-vsphere/blob/master/manifests/controller-manager/vsphere-cloud-controller-manager-ds.yaml#L64

### Risks and Mitigations

#### Specific CCM doesn’t work properly on OpenShift

The `CCMs` have not been tested on OCP yet. Unknown issues may arise.

Severity: medium-high
Likelihood: medium

#### OpenShift components can't work with the CCM properly

Since OpenShift imposes additional limitations compared to Kubernetes, some collisions are possible.
So far there are no CCMs available in OpenShift, and there can be some problems with operability.

Severity: medium-high
Likelihood: low

#### CSI driver is not available in OpenShift

CCM doesn't support the in-tree `Volume` manager implementation. A CSI driver now becomes a mandatory requirement for every out-of-tree provider. Additionally the CSI should support data migration between the in-tree storage solution and the plugin provided storage interfaces.

Severity: medium-high
Likelihood: low

### Test Plan

- Make sure the `CCM` implementation for "any" cloud is up and running in a bootstrap phase before or shortly after the `kubelet` is running on any `Node`.
- Ensure that transition between the in-tree and the out-of-tree will be handled by the established architecture with the new release payload, where all of the existing `Nodes` are operational during and after the upgrade time, and the bootstrap procedure for new ones is successful.
- Ensure that upgrade from a release running on the in-tree provider only to the out-of-tree succeeds, and the downgrade is also supported.
- Each provider should have a working CI implementation, which will assist in testing the provisioning and upgrades with out-of-tree support enabled.

### Graduation Criteria

#### Tech preview

Operator handles OpenStack installation in 4.7.

#### Tech Preview -> GA

The operator will handle basic tasks, such as create initial templates at bootstrap and running all of supported in-tree providers after installation.

## FAQ

Q: Can we disaster-recover a cluster without CCM running?
A: We can't. Basically, without CCM running, nodes can only join the cluster, but they will be unschedulable. Which means it's impossible to start any workloads on the nodes. [Source](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/#running-cloud-controller-manager)

Q: What is non-functional in a cluster during bootstrapping until CCM is up?
A: If CCM is not available, new nodes in the cluster will be left unschedulable.

Q: Who talks to CCM?
A: From an architectural point of view, only Kube API server is able to communicate with CCM. [Source](https://kubernetes.io/docs/concepts/architecture/cloud-controller/#design). All requests to CCM are sent by `kubelet`. KCM doesn't interact with CCM.

Q: CCM relation to other openshift components, such as SDN and storage? How a non-operational CCM will affect cluster health, which components will take a hit?
A: A couple of components rely on CCM at different time of the cluster lifetime:

- `kubelet` dependency requires removal of taints from newly created Nodes.

- Load Balancer setup for `Service` resources is [supported](https://github.com/kubernetes/cloud-provider/blob/master/controllers/service/controller.go) by the CCM interface with the [LoadBalancer()](https://github.com/kubernetes/cloud-provider/blob/0dc0419ec04907f9deb6aa498cece84f10044726/cloud.go#L49] method. The [ingress-operator](https://github.com/openshift/cluster-ingress-operator) is affected by this. Any `Route` resource changes will not be configured while the `CCM` is down, and any request to provision an external access to the cluster will not be satisfied.
  
- Storage CSI migration will be required for OpenStack and vSphere providers

Q: Should we reuse the existing cloud provider config or generate a new one?
A: CCM config is backward compatible with the in-tree cloud provider. It means we can reuse it.

Q: How to migrate PVs created by the in-tree cloud provider?
A: [CSIMigration](https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/) looks like the best option, especially if it is GA in 1.19.

Q: Does CCM provide an API? How is it hosted? HTTPS endpoint? Through load balancer?
A: No. The CCM vendors part of the code serving cloud features from the [cloud-provider](https://github.com/kubernetes/cloud-provider). CCM provides a secure TCP connection on port [10258](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cloud-provider/ports.go#L22) and communicates similarly to other kubernets components.

Q: What happens if the kcm leader has no access to ccm? Can it continue its work? Will it give up leadership?
A: Expect KCM perform independently to CCM and do not care about its state after initial code migration. The CCM cluster operator will be responsible for going Unavailable or Degraded in a similar situation.

Q: Can we disaster-recover a cluster without ccm running?
A: Yes we can, only the Node management and Route provisioning will be compromized. 

Q: What are the thoughts about certificate management?
A: Certificate management should be performed by the CCM operator, like it's done in the KCM operator.

Q: What happens if the KCM leader has no access to CCM? Can it continue its work? Will it give up leadership?
A: Since KCM does not communicate with CCM it can continue to work if CCM is not available.

Q: Does every node need a CCM?
A: No. Control plane nodes only. [Source](https://kubernetes.io/docs/concepts/overview/components/#cloud-controller-manager)

Q: How does SDN depend on CCM?
A: CCMs manage `LoadBalancer` `Service` resources in all in-tree cloud implementations, except vSphere, so their creation will not be possible while the CCM is down. This will affect the [ingress-operator](https://github.com/openshift/cluster-ingress-operator). Cloud networking across `Nodes` is [disabled](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/8be94db9da523152af2268bd7f891fe089a424eb/bindata/v4.1.0/config/defaultconfig.yaml#L8-L9) by default for all clouds, so the routes will not be provisioned for the Node by the `CCM` (which is done by `Routes()` method in the interface), and so not required for SDN functionality.

Q: How metrics are affected by the CCM migration?
A: Currently the `CCM` implementation is using the same set of metrics from the core repository. The `CCMO` will require prometheus configuration to collect them from a different location.

## Other notes

### Removing a deprecated feature

Upstream is currently planning to remove support for the in-tree providers with OCP 4.8 (kubernetes 1.21) - based on the slack [conversation](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600). The enhancement [describes](https://github.com/kubernetes/enhancements/blob/473d6a094f07e399079d1ce4698d6e52ddcc1567/keps/sig-cloud-provider/20190125-removing-in-tree-providers.md) steps towards this process. Our goal is to have a working alternative for the in-tree providers ready to be used on-par with old implementation before the upstream release will remove this feature from core Kubernetes repository.

The `cloud-provider` and `cloud-config` flags from `kubelet` will be removed in 1.23 - [link](https://github.com/kubernetes/kubernetes/pull/90408).

## Timeline

### Upstream

Follow the Kubernetes community discussions and implementation/releases

- vSphere support has graduated to beta in [v1.18](https://github.com/kubernetes/enhancements/issues/670)

- Azure support goes into beta in [v1.20](https://github.com/kubernetes/enhancements/issues/667)

### OpenShift

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

## Additional Links

- [Kubernetes Cloud Controller Managers](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)

- [Cloud Controller Manager Administration](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)

- [The Kubernetes Cloud Controller Manager](https://medium.com/@m.json/the-kubernetes-cloud-controller-manager-d440af0d2be5) article
