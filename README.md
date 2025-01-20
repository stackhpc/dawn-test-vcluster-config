# vcluster-fluxcd-config

This repository contains configuration for deploying and managing one or more
[vClusters](https://www.vcluster.com/docs) on a host [Kubernetes](https://kubernetes.io/) cluster
using a [GitOps](https://about.gitlab.com/topics/gitops/) workflow. The management of the host
cluster is beyond the scope of this repository.

By default, Kubernetes does not provide strong multi-tenancy guarantees. In particular,
[custom resource definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/),
which are being increasingly adopted, are cluster scoped, meaning that in practice, tenants
cannot be permitted to access CRDs without risk of interference with each other.

vCluster allows multiple "virtual" Kubernetes clusters to be deployed on a single host Kubernetes
cluster. Broadly speaking, these clusters take the form of a dedicated Kubernetes control plane,
with its own storage, and a "syncer" that is responsible for synchronising low-level resources
from the virtual cluster to the underlying cluster, e.g. pods and services.

Each vCluster runs inside a namespace on the host cluster, and pods created by the syncer are
created in that namespace on the host cluster regardless of their namespace in the vCluster.
This means that vClusters can be
[isolated from each other](https://www.vcluster.com/docs/vcluster/deploy/topologies/isolated-workloads)
using Kubernetes features such as
[pod security standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/),
[resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/),
[limit ranges](https://kubernetes.io/docs/concepts/policy/limit-range/) and
[network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

For more information, see the [vCluster documentation](https://www.vcluster.com/docs).

## Host cluster

To provision vClusters using this repository, you must first have access to a host cluster.

The host cluster must have a CNI that supports network policies (e.g. Cilium or Calico) and a
[storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/) that can be used
to provision the storage volume for each vCluster (ideally backed by SSD).

This repository uses [ingresses](https://kubernetes.io/docs/concepts/services-networking/ingress/)
to be expose the API servers for the vClusters (although
[other options are available](https://www.vcluster.com/docs/vcluster/manage/accessing-vcluster#expose-vcluster)).
This requires that the host cluster is running an
[ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
with SSL passthrough enabled.

This repository assumes that [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) is being
used to provide ingress, and uses the corresponding annotations to configure SSL passthrough.

> [!WARNING]
> 
> SSL passthrough is **not** enabled by default when deploying `ingress-nginx`. To enable it when
> using the Helm chart, use the following values:
>
> ```yaml
> controller:
>   extraArgs:
>     enable-ssl-passthrough: "true"
> ```

## Usage
