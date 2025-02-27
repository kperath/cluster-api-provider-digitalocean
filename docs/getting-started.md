# Getting started

## Prerequisites

- A [DigitalOcean][DigitalOcean] Account
- Install [clusterctl][clusterctl]
- Install [kubectl][kubectl]
- Install [kustomize][kustomize] `v3.1.0+`
- [Packer][Packer] and [Ansible][Ansible] to build images
- Make to use `Makefile` targets
- A management cluster. You can use either a VM, container or existing Kubernetes cluster as management cluster.
   - If you want to use a VM, install [Minikube][Minikube] version 0.30.0 or greater. You'll also need to install the [Minikube driver][Minikube Driver]. For Linux, we recommend `kvm2`. For MacOS, we recommend `VirtualBox`.
   - If you want to use a container you'll need to install [Kind][kind].
   - If you want to use an existing Kubernetes cluster you'll need to prepare a kubeconfig for the cluster you intend to use.
- Install [doctl][doctl] (optional)

## Feb 27, 2025 Demo setup
```sh
# Install clusterctl
curl -LO https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.9.5/clusterctl-linux-amd64

# Install kubectl
curl -LO https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl

# Install kustomize
 curl -LO https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.6.0/kustomize_v5.6.0_linux_amd64.tar.gz

# Move these binaries after unzipping & archiving them into /usr/local/bin

# Install Packer
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install packer

# Install Ansible
apt-get install pipx
pipx install --include-deps ansible
pipx ensurepath

# Install Docker
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Install Kind
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-$(uname)-amd64

#Install doctl
sudo snap install doctl

# Install Make (Needed by imagebulider)
apt install make

# Install Unzip (Needed by imagebulider)
apit install unzip

# Move all binaries into /usr/local/bin and source your shell config
source ~/.bashrc
```


## Setup Environment

```bash
# Export the DigitalOcean access token
$ export DIGITALOCEAN_ACCESS_TOKEN=<access_token>

# Init doctl
$ doctl auth init --access-token ${DIGITALOCEAN_ACCESS_TOKEN}
```

## Building images

Clone the image builder repository if you haven't already:

    $ git clone https://github.com/kubernetes-sigs/image-builder.git

Change directory to images/capi within the image builder repository:

    $ cd image-builder/images/capi

Choose a DigitalOcean image build target from the list returned by `make | grep build-do` and generate a DigitalOcean image (choosing Ubuntu in the example below):

    $ make build-do-ubuntu-2204

Verify that the image is available in your account and remember the corresponding image ID:

    $ doctl compute image list-user


## Initialize the management cluster

```bash

# with kind cluster do
kind create cluster

$ export DO_B64ENCODED_CREDENTIALS="$(echo -n "${DIGITALOCEAN_ACCESS_TOKEN}" | base64 | tr -d '\n')"

# Initialize a management cluster with digitalocean infrastructure provider.
$ clusterctl init --infrastructure digitalocean


# installs CAPI crds and CAPDO crds and also creates CAPI & CAPDO namespaces

```

The output will be similar to this:

```bash
Fetching providers
Installing cert-manager Version="v0.16.1"
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.3.11" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.11" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.11" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-digitalocean" Version="v0.4.0" TargetNamespace="capdo-system"

Your management cluster has been initialized successfully!

You can now create your first workload cluster by using:

  clusterctl config cluster [name] --kubernetes-version [version] | kubectl apply -f -

```

## Creating a workload cluster

Setting up environment variables:

```bash
$ export DO_REGION=<region>
$ export DO_SSH_KEY_FINGERPRINT=<your-ssh-key-fingerprint> # md5 hash
$ export DO_CONTROL_PLANE_MACHINE_TYPE=<droplet-size>
$ export DO_CONTROL_PLANE_MACHINE_IMAGE=<image-id> # created in the step above. # doctl compute image list-user
$ export DO_NODE_MACHINE_TYPE=<droplet-size>
$ export DO_NODE_MACHINE_IMAGE=<image-id> # created in the step above. # doctl compute image list-user
```

Generate templates for creating workload clusters:

```bash
$ clusterctl generate cluster capdo-quickstart \
    --infrastructure digitalocean \
    --kubernetes-version v1.31.4 \
    --control-plane-machine-count 1 \
    --worker-machine-count 3 > capdo-quickstart-cluster.yaml
```

*You may need to inspect and make some changes to the generated template.*

Create the workload cluster on the management cluster:

```bash
$ kubectl apply -f capdo-quickstart-cluster.yaml
```

You can see the workload cluster resources by using:

```bash
$ kubectl get cluster-api
```

> Note: The control planes won’t be ready until you install the CNI and DigitalOcean Cloud Controller Manager.

To verify that the first control plane is up, use:

```bash
$ kubectl get kubeadmcontrolplane

NAME                                                                               INITIALIZED   API SERVER AVAILABLE   VERSION    REPLICAS   READY   UPDATED   UNAVAILABLE
kubeadmcontrolplane.controlplane.cluster.x-k8s.io/capdo-quickstart-control-plane   true                                 v1.17.11   1                  1         1
```

After the first control plane node has the `initialized` status, you can retrieve the workload cluster's Kubeconfig:

```bash
$ clusterctl get kubeconfig capdo-quickstart > capdo-quickstart.kubeconfig
```

You can verify what kubernetes nodes exist in the workload cluster by using:

```bash
$ KUBECONFIG=capdo-quickstart.kubeconfig kubectl get node

NAME                                   STATUS     ROLES    AGE     VERSION
capdo-quickstart-control-plane-pt926   NotReady   master   10m     v1.17.11
capdo-quickstart-md-0-2vnwv            NotReady   <none>   5m31s   v1.17.11
capdo-quickstart-md-0-5295f            NotReady   <none>   5m30s   v1.17.11
capdo-quickstart-md-0-pm8np            NotReady   <none>   5m28s   v1.17.11
```

### Deploy CNI

Calico is used here as an example.

```bash
$ KUBECONFIG=capdo-quickstart.kubeconfig kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Deploy DigitalOcean CCM and CSI

```bash
# Create digitalocean secret
$ KUBECONFIG=capdo-quickstart.kubeconfig kubectl create secret generic digitalocean --namespace kube-system --from-literal access-token=$DIGITALOCEAN_ACCESS_TOKEN

# Deploy DigitalOcean Cloud Controller Manager
$ KUBECONFIG=capdo-quickstart.kubeconfig kubectl apply -f https://raw.githubusercontent.com/digitalocean/digitalocean-cloud-controller-manager/refs/heads/master/releases/digitalocean-cloud-controller-manager/v0.1.59.yml

# Deploy DigitalOcean CSI (optional)
$ KUBECONFIG=capdo-quickstart.kubeconfig kubectl apply -f https://raw.githubusercontent.com/digitalocean/csi-digitalocean/master/deploy/kubernetes/releases/csi-digitalocean-v1.3.0.yaml
```

After the [CNI](https://github.com/containernetworking/cni) and the [CCM](https://github.com/digitalocean/digitalocean-cloud-controller-manager) have deployed your workload cluster nodes should be in the `ready` state. You can verify this by using:

```bash
$ KUBECONFIG=capdo-quickstart.kubeconfig kubectl get node

NAME                                   STATUS   ROLES    AGE   VERSION
capdo-quickstart-control-plane-pt926   Ready    master   25m   v1.17.11
capdo-quickstart-md-0-2vnwv            Ready    <none>   21m   v1.17.11
capdo-quickstart-md-0-5295f            Ready    <none>   21m   v1.17.11
capdo-quickstart-md-0-pm8np            Ready    <none>   21m   v1.17.11
```

## Deleting a workload cluster

You can delete the workload cluster from the management cluster using:

```bash
$ kubectl delete cluster capdo-quickstart
```

<!-- References -->
[kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[kustomize]: https://github.com/kubernetes-sigs/kustomize/releases
[kind]: https://github.com/kubernetes-sigs/kind#installation-and-usage
[doctl]: https://github.com/digitalocean/doctl#installing-doctl
[Minikube]: https://kubernetes.io/docs/tasks/tools/install-minikube/
[Minikube Driver]: https://minikube.sigs.k8s.io/docs/drivers
[Packer]: https://www.packer.io/intro/getting-started/install.html
[Ansible]: https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html
[DigitalOcean]: https://cloud.digitalocean.com/
[clusterctl]: https://github.com/kubernetes-sigs/cluster-api/releases
[CNI]: https://github.com/containernetworking/cni
[CCM]: https://github.com/digitalocean/digitalocean-cloud-controller-manager
