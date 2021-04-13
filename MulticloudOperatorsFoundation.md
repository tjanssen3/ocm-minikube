# MulticloudOperatorsFoundation.md
This document describes how to install a hub and managed cluster on a Kind instance. See the main [README.md](README.md) for instructions on setup. `multicloud-operators-foundation` is required to get the `ManagedClusterView` resource working. Beyond the initial setup, application and cluster management should be the same as the main document. Note that during these instructions, you'll set up `registration-operator` as well.

## Known Issues
1. `ocm-minikube/registration-operator.diff` does not apply:
```bash
error: patch failed: Makefile:158
error: Makefile: patch does not apply
```
Solution: revert to "known good" revision of `registration-operator`: `daf5b5cf51a7da18243c6f99cee94f8197f53b90`

## Hub install with multicloud-operators-foundation
Reference: https://open-cluster-management.io/getting-started/install-hub/

**NOTE:** Hub needs more CPUs (or at least till we can reduce all replica sets to a single replica)

Start hub cluster (if applicable): `minikube start --profile=hub --cpus=4`

**NOTE:** Need to recheck if the install of `clusters.clusterregistry.k8s.io` CRD is still required

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/cluster-registry/master/cluster-registry-crd.yaml`

Clone `multicloud-operators-foundation`: `git clone git@github.com:open-cluster-management/multicloud-operators-foundation.git`

Apply `multicloud-operators-foundation` diff: `cd ./multicloud-operators-foundation; git apply ../ocm-minikube/multicloud-operators-foundation.diff`

Manually clone `registration-operator` into `multicloud-operators-foundation`: `git clone git@github.com:open-cluster-management/registration-operator.git`

Apply diff for `registration-operator`: `cd registration-operator; git apply ../../ocm-minikube/registration-operator.diff`. This is technically optional at this point because the diff is only required for managed clusters, not the hub.

Run hub deployment script (from `multicloud-operators-foundation` root): `make deploy-hub`

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

### Managed cluster install after hub
Requires hub installed via multicloud-operators-foundation (called `hub` in this example). Example managed cluster is `cluster1`. 

Start new cluster (if applicable): `minikube start --profile=cluster1`

If it doesn't exist yet, clone `registration-operator` into `multicloud-operators-foundation`: `cd multicloud-operators-foundation; git clone git@github.com:open-cluster-management/registration-operator.git`

If `registration-operator` diff isn't applied, do that now: `cd registration-operator; git apply ../../ocm-minikube/registration-operator.diff`. Note this configuration has `ocm-minikube` and `multicloud-operators-foundation` in the same directory.

If `multicloud-operator` diff isn't applied, do that now: `cd ..; git apply ../ocm-minikube/multicloud-operators-foundation.diff`

Set up kubeconfig files and environment variables for your hub and managed cluster:
```bash
# set up hub variables, kubeconfig
kubectl config view --flatten --context=hub --minify > /tmp/hub-config
cp /tmp/hub-config registration-operator/.kubeconfig  # while in multicloud-operators-foundation root
export HUB_KIND_KUBECONFIG=/tmp/hub-config

# set up managed cluster variables, kubeconfig
export KIND_CLUSTER=cluster1
kubectl config view --flatten --context=cluster1 --minify > /tmp/cluster1-config
export KLUSTERLET_KIND_KUBECONFIG=/tmp/cluster1-config
```

Change context to managed cluster and run deploy command: `kubectl config use-context cluster1; make deploy-klusterlet`

Once installation is done, wait for deployments to appear and become ready:
```bash
$ kubectl get deployments -n open-cluster-management
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
klusterlet                   3/3     3            3           80m
klusterlet-registry-server   1/1     1            1           81m

$ kubectl get deployments -n open-cluster-management-agent
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
klusterlet-registration-agent   3/3     3            3           80m
klusterlet-work-agent           3/3     3            3           80m
```

At this point, the managed cluster is installed, but it needs to be registered with the hub to complete the installation. 

Switch context to hub: `kubectl config use-context hub`

Get certificates: `kubectl get csr`:
```
NAME             AGE   SIGNERNAME                                    REQUESTOR          CONDITION
cluster1-cmhj7   41m   kubernetes.io/kube-apiserver-client           minikube-user      Pending
```

Approve the certificate: `kubectl certificate approve cluster1-cmhj7`

Apply patch to accept managedcluster: `kubectl patch managedcluster cluster1 -p='{"spec":{"hubAcceptsClient":true}}' --type=merge`

Verify managed clusters have been accepted: `kubectl get managedcluster`

```bash
Sample output:
NAME       HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
cluster1   true           https://localhost      True     True        3m54s
```

## Enable ManagedClusterView resource
By default, the ManagedClusterView resource is not available through `registration-operator`:

```bash
$ kubectl get managedclusterview
error: the server doesn't have a resource type "managedclusterview" 
```

To remedy this, first apply the crd to hub from `multicloud-operator-foundation` root: `kubectl config use-context hub; kubectl apply -f deploy/foundation/hub/resources/crds/view.open-cluster-management.io_managedclusterviews.yaml`

Then apply a deployment. Note that the default deploys to the `cluster1` namespace: `kubectl apply -f examples/view/getdeployment.yaml`

Now verify it works: `kubectl get managedclusterview --namespace=cluster1`
```bash
NAME            AGE
getdeployment   5m40s
```
