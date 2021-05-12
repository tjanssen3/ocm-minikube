# MulticloudOperatorsFoundation.md
This document describes how to install a hub and managed cluster on a Minikube or Kind instance. See the main [README.md](README.md) for instructions on setup. `multicloud-operators-foundation` is required to get the `ManagedClusterView` resource working. Beyond the initial setup, application and cluster management should be the same as the main document. Note that during these instructions, you'll set up `registration-operator` as well.

The instructions below should be all the steps you need to take to get multicloud-operators-foundation hub and managed cluster running, along with a working ManagedClusterView resource. The instructions are shown for Kind. In comments, when appropriate, there are Minikube-equivalent instructions. 

# Setup
You'll need to apply a diff for Minikube environment setup to work. By default, `multicloud-operators-foundation` expects a Kind setup. The diff will remove the hardcoded `kind-` prefix from registration-operators/Makefile.

## Requirements
1. Kind is installed: [instructions here](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
2. Kustomize is installed: [instructions here](https://kubectl.docs.kubernetes.io/installation/kustomize/)

## Environment Variables You'll Need
By default, this process uses a hub cluster names `hub` and a managed cluster named `cluster1`. If you need to change these names for any reason, you'll need to update environment variables to support the new names. If you apply the `registration-operator` diff, this will include Kind clusters. Examples shown below:

```bash
# cluster names
export HUB_CLUSTER=kind-hub
export MANAGED_CLUSTER=kind-cluster1

# kubeconfig locations
export KLUSTERLET_KIND_KUBECONFIG=~/cluster1-kubeconfig  # update to new cluster info location
export HUB_KIND_KUBECONFIG=~/hub-kubeconfig  # update to new hub kubeconfig location
```

## Prerequisite Setup for multicloud-operators-foundation
### Hub setup
Full documentation here: https://open-cluster-management.io/getting-started/core/cluster-manager/

```bash
# set up hub through Kind
kind create cluster --name hub  # minikube start --profile=hub --cpus=4
kind get kubeconfig --name hub --internal > ~/hub-kubeconfig  # kubectl config view --flatten --context=hub --minify > ~/hub-kubeconfig

git clone https://github.com/open-cluster-management/multicloud-operators-foundation.git  # "ideal" path: ~/go/src/github.com/open-cluster-management

# set up registration operator
cd multicloud-operators-foundation
git clone https://github.com/open-cluster-management/registration-operator
kubectl config use-context kind-hub  # kubectl config use-context hub

cd registration-operator

# apply diff so Minikube setup is supported
git apply ../../ocm-minikube/registration-operator.diff

make deploy-hub
```

### Managed Cluster setup
Full documentation here: https://open-cluster-management.io/getting-started/core/register-cluster/

```bash
# set up cluster1 through Kind
kind create cluster --name cluster1  # minikube start --profile=cluster1
kind get kubeconfig --name cluster1 --internal > ~/cluster1-kubeconfig  # kubectl config view --flatten --context=cluster1 --minify > ~/cluster1-kubeconfig 

kubectl config use-context kind-cluster1  # kubectl config use-context cluster1
export KLUSTERLET_KIND_KUBECONFIG=~/cluster1-kubeconfig
export HUB_KIND_KUBECONFIG=~/hub-kubeconfig
make deploy-spoke-kind
```

### Finish cluster registration
```bash
# apprive CSR on hub cluster
kubectl config use-context kind-hub  # kubectl config use-context hub
kubectl get csr  # make sure csr shows approved, issued AND cluster1 is Pending
MANAGED_CLUSTER=$(kubectl get managedclusters | grep cluster | awk '{print $1}')
CSR_NAME=$(kubectl get csr |grep $MANAGED_CLUSTER | grep Pending |awk '{print $1}')
kubectl certificate approve "${CSR_NAME}"

# accept managed cluster on hub
MANAGED_CLUSTER=$(kubectl get managedclusters | grep cluster | awk '{print $1}')
kubectl patch managedclusters $MANAGED_CLUSTER  --type merge --patch '{"spec":{"hubAcceptsClient":true}}'

kubectl get managedclusters  # should see cluster1 listed as Hub Accepted, JOINED True and AVAILABLE True
```

## Install multicloud-operators-foundation
```bash
kubectl config use-context kind-hub  # kubectl config use-context hub

cd ..  # pwd=multicloud-operators-foundation for the following steps

make deploy-foundation-hub
export MANAGED_CLUSTER_NAME=cluster1
make deploy-foundation-agent
```

## Notes
1. Kind references are hard-coded in [registration-operator](https://github.com/open-cluster-management/registration-operator). The diff removes `kind-` references from Makefile. For full compatibility, use `export HUB_CLUSTER=hub` and `export MANAGED_CLUSTER=cluster1` before installing. 
