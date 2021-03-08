# Open-cluster-management(OCM) with minikube

**NOTE:** Currently published OCM OLM bundles are outdated, hence install is via sources

The write-up is presented as a series of commands that need to be executed. This includes a series of `kubectl config use-context <cluster-name>` which assumes commands following the same are not changing the context and are retained till the next context change.

There are 3 diffs present with this document, which need to be applied to the repositories as follows,
- `registration-operator.diff` to git@github.com:open-cluster-management/registration-operator
- `multicloud-operators-subscription.diff` to git@github.com:open-cluster-management/multicloud-operators-subscription.git
- `application-samples.diff` to git@github.com:open-cluster-management/application-samples.git

## Pre-requisites

1) Use `kvm2` as the minikube driver for the setup

    NOTE: `docker` driver was found not to work correctly for this setup!

## Setup

1) minikube cluster with profile `hub` serves as the OCM hub
2) minikube cluster on the same node with profile `cluster1` serves as one of the managed clusters
3) **TODO:** Add hub as another managed cluster

## Hub install

Reference: https://open-cluster-management.io/getting-started/install-hub/

**NOTE:** Hub needs more CPUs (or at least till we can reduce all replica sets to a single replica)

`minikube start --profile=hub --cpus=4`

**NOTE:** Need to recheck if the install of `clusters.clusterregistry.k8s.io` CRD is still required

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/cluster-registry/master/cluster-registry-crd.yaml`

`git clone git@github.com:open-cluster-management/registration-operator.git`

`cd ./registration-operator`

`git apply ../ocm-minikube/registration-operator.diff`

`make deploy-hub`

```
Pods at the end of hub deployment:
olm                           catalog-operator-54bbdffc6b-k22xq                          1/1     Running   0          2m8s
olm                           olm-operator-6bfbd74fb8-ltz5g                              1/1     Running   0          2m8s
olm                           operatorhubio-catalog-v4n9g                                1/1     Running   0          119s
olm                           packageserver-594555d95d-mpsnv                             1/1     Running   0          118s
olm                           packageserver-594555d95d-t9znd                             1/1     Running   0          118s
open-cluster-management-hub   cluster-manager-registration-controller-566b5cd967-9dwbg   1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-registration-controller-566b5cd967-jzqgk   1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-registration-controller-566b5cd967-pm6tp   1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-registration-webhook-5649df75cf-26l8d      1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-registration-webhook-5649df75cf-lp4w7      1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-registration-webhook-5649df75cf-r8k55      1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-work-webhook-776d978d64-2rxw2              1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-work-webhook-776d978d64-hxr8w              1/1     Running   0          57s
open-cluster-management-hub   cluster-manager-work-webhook-776d978d64-jkxz4              1/1     Running   0          57s
open-cluster-management       cluster-manager-5446b78b75-5r4sm                           1/1     Running   0          84s
open-cluster-management       cluster-manager-5446b78b75-9l5tz                           1/1     Running   0          84s
open-cluster-management       cluster-manager-5446b78b75-bwdbs                           1/1     Running   0          84s
open-cluster-management       cluster-manager-registry-server-84fbf889b4-npr2w           1/1     Running   0          102s
```

## Managed Cluster install

**NOTE: Repeat the steps to add another managed cluster including the hub itself**

### Deploy managed cluster components

Reference: https://open-cluster-management.io/getting-started/register-cluster/#install-from-source

`minikube start --profile=cluster1`

`kubectl config view --flatten --context=hub --minify > /tmp/hub-config`

`kubectl config view --flatten --context=cluster1 --minify > /tmp/cluster1-config`

`kubectl config use-context cluster1`

`export KLUSTERLET_KIND_KUBECONFIG=/tmp/cluster1-config`

`export KIND_CLUSTER=cluster1`

`export HUB_KIND_KUBECONFIG=/tmp/hub-config`

`make deploy-spoke-kind`

**NOTE:** Wait for deployments to appear

```
Sample output:
$ kubectl get deployments -n open-cluster-management
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
klusterlet                   3/3     3            3           80m
klusterlet-registry-server   1/1     1            1           81m

$ kubectl get deployments -n open-cluster-management-agent
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
klusterlet-registration-agent   3/3     3            3           80m
klusterlet-work-agent           3/3     3            3           80m
```

### Complete managed cluster registration on hub

Reference: https://open-cluster-management.io/getting-started/register-cluster/#what-is-next

`kubectl config use-context hub`

`kubectl get csr`

```
Sample output:
NAME             AGE    SIGNERNAME                                    REQUESTOR         CONDITION
cluster1-2bcsp   102s   kubernetes.io/kube-apiserver-client           minikube-user     Pending
csr-wwvpf        34m    kubernetes.io/kube-apiserver-client-kubelet   system:node:hub   Approved,Issued
```

`kubectl certificate approve cluster1-2bcsp`

`kubectl get csr`

```
Sample output:
NAME             AGE    SIGNERNAME                                    REQUESTOR         CONDITION
cluster1-2bcsp   2m2s   kubernetes.io/kube-apiserver-client           minikube-user     Approved,Issued
csr-wwvpf        35m    kubernetes.io/kube-apiserver-client-kubelet   system:node:hub   Approved,Issued
```

`kubectl get managedcluster`

```
Sample output:
NAME       HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
cluster1   false          https://localhost                           3m15s
```

`kubectl patch managedcluster cluster1 -p='{"spec":{"hubAcceptsClient":true}}' --type=merge`

`kubectl get managedcluster`

```
Sample output:
NAME       HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
cluster1   true           https://localhost      True     True        3m54s
```

### Test

Reference: https://open-cluster-management.io/getting-started/register-cluster/#what-is-next

`kubectl config use-context hub`

NOTE: Edit kustomize to change namespace to $KIND_CLUSTER

`sed -e "s,KIND_CLUSTER,$KIND_CLUSTER," -i ../ocm-minikube/examples/kustomization.yaml`

`kubectl apply -k ../ocm-minikube/examples/`

- Tidy up!
`sed -e "s,$KIND_CLUSTER,KIND_CLUSTER," -i ../ocm-minikube/examples/kustomization.yaml`

`kubectl get pods --context=cluster1`

```
Sample output:
NAME    READY   STATUS    RESTARTS   AGE
hello   1/1     Running   0          23s
```

## Application management install

### Hub operator
Reference: https://open-cluster-management.io/getting-started/install-application/#install-from-source

`git clone git@github.com:open-cluster-management/multicloud-operators-subscription.git`

`cd multicloud-operators-subscription`

`git apply ../ocm-minikube/multicloud-operators-subscription.diff`

`kubectl config use-context hub`

`USE_VENDORIZED_BUILD_HARNESS=faked make deploy-community-hub`

**NOTE:** Faked `USE_VENDORIZED_BUILD_HARNESS`, does not impact the make target in use above

**NOTE:** Wait for deployments to appear

```
Sample output:
kubectl get deployments -n multicluster-operators
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
multicluster-operators-application    1/1     1            1           47s
multicluster-operators-subscription   1/1     1            1           47s
```

#### ManagedCluster operator for the HUB

**NOTE: This assumes hub is already added as a ManagedCluster following this [section](#managed-cluster-install), replacing `cluster1` with `hub`**

`kubectl config use-context hub`

Check if `hub` is available as a ManagedCluster,

`kubectl get managedcluster hub`

**IMPORTANT: Label the hub ManagedCluster as local, in the example below the hub MC was named `hub` replace as required based on the name hub was added as a MC**
    - `export MANAGED_CLUSTER_NAME=hub`
    - `kubectl patch managedcluster ${MANAGED_CLUSTER_NAME} -p='{"metadata":{"labels":{"local-cluster":"true"}}}' --type=merge`

`cd ../multicloud-operators-subscription`

`export HUB_KUBECONFIG=/tmp/hub-config`

`cp -f ${HUB_KUBECONFIG} /tmp/kubeconfig`

`kubectl -n multicluster-operators create secret generic appmgr-hub-kubeconfig --from-file=kubeconfig=/tmp/kubeconfig`

`mkdir -p munge-manifests`

`cp deploy/managed/operator.yaml munge-manifests/operator.yaml`

`sed -i 's/<managed cluster name>/'"$MANAGED_CLUSTER_NAME"'/g' munge-manifests/operator.yaml`

`sed -i 's/<managed cluster namespace>/'"$MANAGED_CLUSTER_NAME"'/g' munge-manifests/operator.yaml`

`sed -i '0,/name: multicluster-operators-subscription/{s/name: multicluster-operators-subscription/name: multicluster-operators-subscription-mc/}' munge-manifests/operator.yaml`

`kubectl apply -f munge-manifests/operator.yaml`

**NOTE:** Wait for deployments to appear

```
Sample output:
kubectl get deployments -n multicluster-operators
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
multicluster-operators-application       1/1     1            1           3h42m
multicluster-operators-subscription      1/1     1            1           3h42m
multicluster-operators-subscription-mc   1/1     1            1           20s
```

### ManagedCluster operator

**NOTE: If adding the hub as one of the managed clusters, see [above](#hub-as-a-managedcluster) instructions instead of following this section!**

`export HUB_KUBECONFIG=/tmp/hub-config`

`kubectl config use-context cluster1`

`export MANAGED_CLUSTER_NAME=cluster1`

`USE_VENDORIZED_BUILD_HARNESS=faked make deploy-community-managed`

**NOTE:** Faked `USE_VENDORIZED_BUILD_HARNESS`, does not impact the make target in use above

**NOTE:** Wait for deployments to appear

```
Sample output:
kubectl get deployments -n multicluster-operators
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
multicluster-operators-subscription   1/1     1            1           17s
```

### Test

`kubectl config use-context hub`

`kubectl apply -f examples/helmrepo-hub-channel`

NOTE: When adding a second MC, there were initial MC namespace not found errors in the hub deployment `multicluster-operators/multicluster-operators-application` logs. Restarting the deployment fixed the issue (which was done to increase log verbosity, but fixed the issue instead)

```
Sample output:
kubectl get pods --context=cluster1
NAME                                                   READY   STATUS    RESTARTS   AGE
hello                                                  1/1     Running   0          25m
nginx-ingress-1619c-controller-5959b6b4c4-fd856        1/1     Running   0          6m52s
nginx-ingress-1619c-default-backend-84f598dd65-jxmkd   1/1     Running   0          6m52s
```

## Application samples test

Label the managed cluster `cluster1` as `usage=test`

`kubectl patch managedcluster cluster1 -p='{"metadata":{"labels":{"usage":"test"}}}' --type=merge`

`git clone git@github.com:open-cluster-management/application-samples.git`

`cd application-samples`

`git apply ../ocm-minikube/application-samples.diff`

`kubectl apply -k subscriptions/channel`

`kubectl apply -k subscriptions/book-import`

One error was noted and application CR is not created, but example still works:
```
kubectl apply -f subscriptions/book-import/application.yaml 
Error from server (InternalError): error when creating "subscriptions/book-import/application.yaml": Internal error occurred: failed calling webhook "applications.apps.open-cluster-management.webhook": Post "https://multicluster-operators-application-svc.multicluster-operators.svc:443/app-validate?timeout=10s": dial tcp 10.111.143.132:443: connect: connection refused
```

```
Sample output:
$ kubectl get pods --context=cluster1 -n book-import
NAME                          READY   STATUS    RESTARTS   AGE
book-import-849fc8cb8-7697x   1/1     Running   0          58m
```
