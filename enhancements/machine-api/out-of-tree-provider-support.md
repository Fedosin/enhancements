---
title: out-of-tree-provider-support
authors:
  - "@Danil-Grigorev"
  - "@mfedosin"
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

This enhancement proposal describes steps needed to add support for out-of-tree providers in cloud providers currently supported by Openshift. This includes: AWS, Azure, GCP, vSphere, Openstack.

As a pioneer platform, it is proposed to use OpenStack. 

OpenStack external cloud provider fixes many issues and limitations with the in-tree cloud provider, such as it's reliance on Nova metadata service. For OpenStack platform, this means the possibility for deploying on provider networks and at the edge.

## Motivation

The Kubernetes community is currently moving towards exclusion of the in-tree cloud-provider code from the core Kubernetes repository, and instead introducing out-of-tree provider support, making them independent from the core Kubernetes release cycle, and simplifying the process of development and e2e testing for those plugins. Previously each cloud-provider was located in the [`legacy-cloud-providers`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/legacy-cloud-providers) and now the support and main development is done in the dedicated cloud-provider [repositories](https://github.com/kubernetes?q=cloud-provider-). They plan to remove the in-tree support completely in the future releases, approximetly in [v1.21](https://kubernetes.slack.com/archives/CPPU7NY8Y/p1589289501020600).

### Goals

- Prepare openshift components to accomodate the out-of-tree plugins and remove support for the in-tree implementation.
- Describe an approach how an `external` cloud provider configuration for the cloud will be supported.
- Provide the means to allow generic `CCM` component registration via the provider specific upstream implemetation.
- Concider support for running switching cloud configurations in a single cluster.

### Non-Goals

- Force immediate exclusion of in-tree support for currently used providers, as their out-of-tree counterparts are added.
- Add support for CSI and other out-of-tree implementations of cluster storage interfaces.

## Proposal

Add an out-of-tree support based on current cluster infrastructure, and make a seamless transition for all currently supported providers.

This change will be gradually executed on all suported cloud providers: OpenStack, AWS, Azure, GCP and vSphere. The change could help inclusion for other cloud providers, such as [Yandex](https://github.com/flant/yandex-cloud-controller-manager) and [IBM](https://github.com/kubernetes/enhancements/issues/671) `CCMs` (`cloud-controller-managers`), if this will be concidered a goal.

### User Stories

#### Story 1:

As a cloud developer, I’d like to improve support for cloud features for openshift. I'd like to develop and test new cloud providers with do not ship with default kubernetes distiribution, and require support for external cloud controller manager.

#### Story 2:

As a developer I'd like to develop, build and release fixes independently from the kubernetes core, and assume they will first land upstream and then will be carried over into openshift destribution with less effort.

#### Story 3:

We’d like to discuss technical details related to a specific cloud in a SIG meeting with people who are also involved into development in this domain, and that way gain useful insights into the cloud infrastructure, improve the overall quality of our features, stay on top of the new features, and improve the relations with maintainers outside of our company, which nevertheless share with us a common goal.

### Implementation Details

The `cloud-controller-manager-operator` implementation will be hosted within openshift repository in https://github.com/openshift/cluster-cloud-controller-manager-operator. That repository will manage `CCM` provisioning for metioned cloud providers.

- `CCM` will be provisioned in the `openshift-cloud-controller-manager` namespace.
- Operator will be deployed in the `openshift-cloud-controller-manager-operator` namespace.

Preparation steps, common for all providers:

- Add support for recognition of the `external` configuration for each provider in [library-go](https://github.com/openshift/library-go/blob/master/pkg/operator/configobserver/cloudprovider/observe_cloudprovider.go).
- Port the templates of `cloud-controller` from upstream repositories into proposed https://github.com/openshift/cluster-cloud-controller-manager-operator repository. Those templates will belong to the newly created `openshift-cloud-controller-manager` namespace and be managed with an operator.

Each provider implementation will need to do the following minimal set of actions in order to make out-of-tree implemetation work:

- Use `external` flag in the `kube-controller-manager` pod instead of the previously used provider. Remove the `cloud-config` option from the template.
- Use `external` flag in the `kubelet` pod. Remove the `cloud-config` option from the template.
- Provision a `cloud-controller-manager` `DaemonSet` in the `openshift-cloud-controller-manager` namespace.

#### Resource changes

##### Pre-release / Development

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

#### Release changes

Once the provider is ready to be published with an extenal `CCM` support, the code parts for in-tree enablement will be removed on [these](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85) lines.

#### Kubelet

Openshift manages `kubelet` configuration with `machine-config-operator`. The actual `cloud-config` arguments are set externally, based on the `Infrastructure` resource values on https://github.com/openshift/machine-config-operator/blob/2a51e49b0835bbd3ac60baa685299e86777b064f/pkg/controller/template/render.go#L310-L357, where the support for `external` cloud provider will be added. For each platform with `external` configuration, the `cloud-config` flag will not be set. 

Kubelet is tightly coupled with `CCMs`, due to its [usage](https://github.com/kubernetes/kubernetes/blame/323f34858de18b862d43c40b2cced65ad8e24052/pkg/kubelet/cloudresource/cloud_request_manager.go#L98-L102) of cloud-provider interfaces, such as `Instances` and `NodeAdresses`. Whenever there is a need to bootstrap any `Node`, `kubelet` will collect missing information about the instance IP addresses, type and zone. This information could later be used by `kube-scheduler`.

During this procedure all uninitialized `Nodes` in the cluster are [tainted](https://github.com/kubernetes/kubernetes/blob/c53ce48d48372c30053a9b67c6d1dee237b5cd69/pkg/kubelet/kubelet_node_status.go#L307-L311) with `node.cloudprovider.kubernetes.io/uninitialized: true`. This taint is removed after the [initialization](https://github.com/kubernetes/kubernetes/blob/46d522d0c8e94edb6d29606a28785e35259badd7/staging/src/k8s.io/cloud-provider/controllers/node/node_controller.go#L374-L376) by the cloud-provider.

#### Kube-controller-manager

This configuration is managed by the `cluster-kube-controller-manager-operator` code and will need to integrate the `library-go` changes, described above in the operator [configuration](https://github.com/openshift/cluster-kube-controller-manager-operator/blob/990ea3f2ace13be6579c00a889842c3f5d3e756a/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L82-L85).

#### Kube-api-server

This component requires same [configuration](https://github.com/openshift/cluster-kube-apiserver-operator/blob/d54e109b333e4db52b681a6979c706b1bdd3a778/pkg/operator/configobservation/configobservercontroller/observe_config_controller.go#L133) changes as `kcm`.

### Bootstrap changes

For the initial transition from in-tree to the out-of-tree, `CCM` pod will be created as static pod on the bootstrap node, to ensure swift removal of the `node.cloudprovider.kubernetes.io/uninitialized: true` taint from a newly created `Nodes`. Later stages, including cluster upgrades will be managed by an operator, which will ensure stability of the configuration, and will run a `CCM` in a `Deployment`.

### Operator resource management strategy

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

Vsphere introduced an additional service to expose internal API server: https://github.com/kubernetes/cloud-provider-vsphere/blob/master/manifests/controller-manager/vsphere-cloud-controller-manager-ds.yaml#L64

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

Each provider will be built with an image of its own, which will be included in the openshift payload. Rule of the thumb: n provider images + operator image.
