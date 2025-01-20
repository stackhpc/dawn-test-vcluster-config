# vcluster-fluxcd-config

This repository contains configuration for deploying and managing one or more
[vClusters](https://www.vcluster.com/docs) on a host [Kubernetes](https://kubernetes.io/) cluster
using a [GitOps](https://about.gitlab.com/topics/gitops/) workflow via [Flux CD](https://fluxcd.io/).
The management of the host cluster is beyond the scope of this repository.

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
vClusters can even be configured to target different nodes in the host cluster using node labels.

For more information, see the [vCluster documentation](https://www.vcluster.com/docs).

## Host cluster

To provision vClusters using this repository, you must first have access to a host cluster.

The host cluster must have a CNI that supports network policies (e.g. Cilium or Calico) and a
[storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/) that can be used
to provision the storage volume for each vCluster (ideally backed by SSD).

### Ingress

This repository uses [ingresses](https://kubernetes.io/docs/concepts/services-networking/ingress/)
to expose the API servers for the vClusters (although
[other options are available](https://www.vcluster.com/docs/vcluster/manage/accessing-vcluster#expose-vcluster)).
This requires that the host cluster is running an
[ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
with SSL passthrough enabled.

This repository assumes that [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) is being
used to provide ingress, and uses the corresponding annotations to configure SSL passthrough.
However it could be easily adapted to use other ingress controllers that allow SSL passthrough.

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

### Flux CD

The host cluster must have the [Flux CD](https://fluxcd.io/) controllers installed.

## Usage

First, fork or copy this repository into a new repository.

To define a new cluster, just copy [the example](./clusters/example/) cluster and modify the
config to suit your use case.

### Configuring the namespace

`namespace.yaml` contains the definition for the namespace that the vCluster will use. Annotations
can be applied to the namespace, e.g. to determine which pod security standard will be used.

The `namespace` in `kustomization.yaml` must be updated to match the name of the namespace in
`namespace.yaml`.

### Configuring the vCluster

`overrides.yaml` contains overrides for the vCluster configuration. The full range of possible
overrides can be found in the
[vCluster docs](https://www.vcluster.com/docs/vcluster/configure/vcluster-yaml/).

In particular, the hostname for the ingress that is used for the API server is required. This
hostname must resolve to the IP address for the ingress controller's load balancer.

The example cluster also includes configuration for authenticating using
[OpenID Connect (OIDC)](https://openid.net/developers/how-connect-works/). In order to configure
this, you must first create an OIDC client with the
[device flow](https://www.oauth.com/oauth2-servers/device-flow/) enabled. This process will differ
for each identity provider and is beyond the scope of this documentation.

> [!NOTE]
> An OIDC client per vCluster is recommended.

The example cluster also includes configuration for binding the `cluster-admin` role _within the
vcluster_ to a group from the OIDC claims. Whether this is suitable for production depends entirely
on your use case.

### Configuring Flux

To add the cluster to the Flux configuration, edit the root `kustomization.yaml` to point to the
cluster directory:

```yaml
resources:
  - ./clusters/cluster1
  - ./clusters/cluster2
```

This will need to be done for each new cluster that is added.

Configuring Flux to manage the vClusters defined in the repository is a one-time operation:

```sh
flux create source git vclusters --url=<giturl> --branch=main
flux create kustomization vclusters --source=GitRepository/vclusters --prune=true
```

This creates a [Kustomization](https://fluxcd.io/flux/components/kustomize/kustomizations/) that
will deploy the root `kustomization.yaml` from this repository, hence deploying all the vClusters
referenced in that file.

## Accessing a vCluster

This section assumes that the vCluster has been configured to use OIDC for authentication. Other
mechanisms for accessing vClusters are described in the
[vCluster documentation](https://www.vcluster.com/docs/vcluster/manage/accessing-vcluster).

To use OIDC to access a vCluster, the client must have the
[oidc-login](https://github.com/int128/kubelogin) plugin for `kubectl` installed.

First, you must obtain the base64-encoded certificate for the vCluster's API server, using a
kubeconfig file that can access the **host** cluster:

```sh
kubectl -n my-vcluster-ns get secret vcluster-certs -o go-template='{{index .data "ca.crt"}}'
```

Then create a `KUBECONFIG` file similar to the following to access the vCluster, replacing the
`server` with the ingress hostname for the API server and the OIDC issuer and client ID with
the values used when configuring the vCluster:

```yaml
apiVersion: v1
clusters:
  - cluster:
      server: https://my-cluster.k8s.example.org:443
      certificate-authority-data: <BASE64-ENCODED CERT DATA>
    name: vcluster
contexts:
  - context:
      cluster: vcluster
      user: oidc
    name: oidc@vcluster
current-context: oidc@vcluster
kind: Config
preferences: {}
users:
  - name: oidc
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --grant-type=device-code
          - --oidc-issuer-url=https://myidp.example.org
          - --oidc-client-id=my-vcluster
```

This configuration will perform a device flow authentication with the issuer to get an OIDC
token that can be used to interact with Kubernetes.
