# Compliant Kubernetes on Openstack

This document contains instructions on how to set up a Compliant Kubernetes environment (consisting of a service cluster and one or more workload clusters) on Openstack.
TODO: The document is split into two parts:

- Cluster setup (setting up infrastructure and the Kubernetes clusters).
  We will be using the [Terraform module for Openstack that can be found in the Kubespray repository](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/terraform/openstack).
  Please refer to it if you need more details about this part of the setup.

- Apps setup (including information about limitations)

Before starting, make sure you have [all necessary tools](getting-started.md). In addition to these general tools, you will also need:

- [Openstack credentials](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/terraform/openstack#openstack-access-and-credentials) (either using `openrc` or the `clouds.yaml` configuration file) for setting up the infrastructure.

## Initialize configuration folder

Choose names for your service cluster and workload cluster(s):

```bash
SERVICE_CLUSTER="testsc"
WORKLOAD_CLUSTERS=( "testwc0" "testwc1" )
```

Start by initializing a Compliant Kubernetes environment using Compliant Kubernetes Kubespray.
All of this is done from the root of the `compliantkubernetes-kubespray` repository.

```bash
export CK8S_CONFIG_PATH=~/.ck8s/<environment-name>
export SOPS_FP=<PGP-fingerprint>

for CLUSTER in "${SERVICE_CLUSTER}" "${WORKLOAD_CLUSTERS[@]}"; do
  ./bin/ck8s-kubespray init "${CLUSTER}" openstack "${SOPS_FP}"
done
```

## Infrastructure setup using Terraform

Configure Terraform by creating a `cluster.tfvars` file for each cluster.
The available options can be seen in `kubespray/contrib/terraform/openstack/variables.tf`.
There is a sample file that can be copied to get something to start from.

```bash
for CLUSTER in ${SERVICE_CLUSTER} ${WORKLOAD_CLUSTERS[@]}; do
  cp kubespray/contrib/terraform/openstack/sample-inventory/cluster.tfvars "${CK8S_CONFIG_PATH}/${CLUSTER}-config/cluster.tfvars"
done
```

!!!note
    You really *must* edit the values in these files.
    There is no way to set sane defaults for what flavor to use, what availability zones or networks are available across providers.

### Expose Openstack credentials to Terraform

Terraform will need access to Openstack credentials in order to create the infrastructure.
More details can be found [here](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/terraform/openstack#openstack-access-and-credentials).
We will be using the declarative option with the `clouds.yaml` file.
Since this file can contain credentials for multiple environments, we specify the name of the one we want to use in the environment variable `OS_CLOUD`:

```bash
export OS_CLOUD=<name-of-openstack-cloud-environment>
```

### Initialize and apply Terraform

```bash
MODULE_PATH="$(pwd)/kubespray/contrib/terraform/openstack"

for CLUSTER in "${SERVICE_CLUSTER}" "${WORKLOAD_CLUSTERS[@]}"; do
  pushd "${CK8S_CONFIG_PATH}/${CLUSTER}-config"
  terraform init "${MODULE_PATH}"
  terraform apply -var-file=cluster.tfvars "${MODULE_PATH}"
  popd
done
```

!!!warning
    The above will not work well if you are using a bastion host.
    This is due to [some hard coded paths](https://github.com/kubernetes-sigs/kubespray/blob/master/contrib/terraform/openstack/modules/compute/main.tf#L207).
    To work around it, you may link the `kubespray/contrib` folder to the correct relative path, or make sure your `CK8S_CONFIG_PATH` is already at a proper place relative to the same.

## Install Kubernetes using Kubespray

Before we can run Kubespray, we will need to go through the relevant variables.
Additionally we will need to expose some credentials so that Kubespray can set up cloud provider integration.

You will need to change at least one value: `kube_oidc_url` in `group_vars/k8s-cluster/ck8s-k8s-cluster.yaml`.

For cloud provider integration, you have a few options [as described here](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/openstack.md#the-in-tree-cloud-provider).
We will be going with the in-tree cloud provider and simply source the Openstack credentials.
This doesn't require any changes to the variables as set up using this guide.

### Run Kubespray

Copy the script for generating dynamic ansible inventories:

```bash
for CLUSTER in "${SERVICE_CLUSTER}" "${WORKLOAD_CLUSTERS[@]}"; do
  cp kubespray/contrib/terraform/terraform.py "${CK8S_CONFIG_PATH}/${CLUSTER}-config/inventory.ini"
  chmod +x "${CK8S_CONFIG_PATH}/${CLUSTER}-config/inventory.ini"
done
```

Now it is time to run the Kubespray playbook!

```bash
for CLUSTER in "${SERVICE_CLUSTER}" "${WORKLOAD_CLUSTERS[@]}"; do
  ./bin/ck8s-kubespray apply "${CLUSTER}"
done
```

### Test access to the Kubernetes API

You should now have an encrypted kubeconfig file for each cluster under `$CK8S_CONFIG_PATH/.state`.
Check that they work like this:

```bash
for CLUSTER in "${SERVICE_CLUSTER}" "${WORKLOAD_CLUSTERS[@]}"; do
  sops exec-file "${CK8S_CONFIG_PATH}/.state/kube_config_${CLUSTER}.yaml" "kubectl --kubeconfig {} cluster-info"
done
```

The output should be similar to this.

```
Kubernetes control plane is running at https://<public-ip>:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
Kubernetes control plane is running at https://<public-ip>:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Prepare for Compliant Kubernetes Apps

To make the kubeconfig files work with Compliant Kubernetes Apps, you will need to rename or copy them, since Compliant Kubernetes Apps currently only support clusters named `sc` and `wc`.
If you have multiple workload clusters, you can make this work by setting `CK8S_CONFIG_PATH` to each `$CK8S_CONFIG_PATH/$CLUSTER-config` in turn.
I.e. `CK8S_CONFIG_PATH` will be different for compliantkubernetes-kubespray and compliantkubernetes-apps.

```
# In compliantkubernetes-kubespray
CK8S_CONFIG_PATH=~/.ck8s/<environment-name>
# In compliantkubernetes-apps (one config path per workload cluster)
CK8S_CONFIG_PATH=~/.ck8s/<environment-name>/<prefix>-config
```

Copy the kubeconfig files to a path that Apps can find.

**Option 1** - A single workload cluster:

```bash
cp "${CK8S_CONFIG_PATH}/.state/kube_config_${SERVICE_CLUSTER}.yaml" "${CK8S_CONFIG_PATH}/.state/kube_config_sc.yaml"
cp "${CK8S_CONFIG_PATH}/.state/kube_config_${WORKLOAD_CLUSTERS[@]}.yaml" "${CK8S_CONFIG_PATH}/.state/kube_config_wc.yaml"
```

You can now use the same `CK8S_CONFIG_PATH` for Apps as for compliantkubernetes-kubespray.

**Option 2** - A multiple workload cluster:

```bash
mkdir -p "${CK8S_CONFIG_PATH}/${SERVICE_CLUSTER}-config/.state"
cp "${CK8S_CONFIG_PATH}/.state/kube_config_${SERVICE_CLUSTER}.yaml" "${CK8S_CONFIG_PATH}/${SERVICE_CLUSTER}-config/.state/kube_config_sc.yaml"
for CLUSTER in "${WORKLOAD_CLUSTERS[@]}"; do
  mkdir -p "${CK8S_CONFIG_PATH}/${CLUSTER}-config/.state"
  cp "${CK8S_CONFIG_PATH}/.state/kube_config_${CLUSTERS}.yaml" "${CK8S_CONFIG_PATH}/${CLUSTER}-config/.state/kube_config_wc.yaml"
done
```

You will then need to set `CK8S_CONFIG_PATH` to each `$CLUSTER-config` folder in turn, in order to install Apps on the service cluster and workload clusters.

With this you should be ready to install Compliant Kubernetes Apps on top of the clusters.
This will be similar to any other cloud provider.
Suggested next steps are:

1. Configure DNS using your favorite DNS provider.
2. Create object storage buckets for backups and container registry storage (if desired).
3. Install Compliant Kubernetes Apps!
   See for example [this guide](../exoscale) for more details.

## Cleanup

If you installed Compliant Kubernetes Apps, start by [cleaning it up](../clean-up).
Make sure you remove all PersistentVolumes and Services with `type=LoadBalancer`.
These objects may create cloud resources that are not managed by Terraform, and therefore would not be removed when we destroy the infrastructure.

Destroy the infrastructure using Terraform, the same way you created it:

```bash
MODULE_PATH="$(pwd)/kubespray/contrib/terraform/openstack"
for CLUSTER in "${SERVICE_CLUSTER}" "${WORKLOAD_CLUSTERS[@]}"; do
  pushd "${CK8S_CONFIG_PATH}/${CLUSTER}-config"
  terraform init "${MODULE_PATH}"
  terraform destroy -var-file=cluster.tfvars "${MODULE_PATH}"
  popd
done
```

Don't forget to remove any DNS records and object storage buckets that you may have created.
