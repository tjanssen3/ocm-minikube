# Open-cluster-management(OCM) with minikube

Base reference for procedure: https://github.com/open-cluster-management/submariner-addon#test-locally-with-kind

## Installing OCM hub
`$ minikube start --profile=cluster1 --cpus=4`

**NOTE:** Hub serves as one of the managed clusters as well

**NOTE:** Hub requires more cpus!

### Installing OLM
Reference: https://olm.operatorframework.io/docs/getting-started/

`export olm_release=v0.17.0`

`$ kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${olm_release}/crds.yaml`

`$ kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${olm_release}/olm.yaml`

**ALERT:** Wait for deployments to be ready before proceeding
```
$ kubectl get deployment -n olm
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
catalog-operator   1/1     1            1           11m
olm-operator       1/1     1            1           11m
packageserver      2/2     2            2           11m
```

### Create operator group for OCM
`$ kubectl create ns open-cluster-management`

`$ kubectl apply -f operatorgroup.yaml`

### Create OCM hub
`$ kubectl apply -f subscription-cluster-manager.yaml`

**ALERT:** Wait for deployments to be ready before proceeding
```
$ kubectl get deployments -n open-cluster-management
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
cluster-manager   3/3     3            3           9m25s
```

Reference: https://github.com/open-cluster-management/registration-operator/blob/master/deploy/cluster-manager/config/samples/operator_open-cluster-management_clustermanagers.cr.yaml

`$ kubectl apply -f clustermanager.yaml`

**ALERT:** Wait for deployments to be ready before proceeding
```
$ kubectl get deployments -n open-cluster-management-hub
NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
hub-cluster-manager-registration-controller   3/3     3            3           6m18s
hub-cluster-manager-registration-webhook      3/3     3            3           6m18s
hub-cluster-manager-work-webhook              3/3     3            3           6m18s
```

### First OCM managed cluster

**NOTE:** We use the OCM hub cluster as one of the managed clusters

Reference: https://github.com/open-cluster-management/registration-operator/blob/master/deploy/klusterlet/config/samples/operator_open-cluster-management_klusterlets.cr.yaml

`$ kubectl apply -f subscription-klusterlet.yml`

**ALERT:** Wait for deployments to be ready before proceeding
```
$ kubectl get deployments -n open-cluster-management klusterlet
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
klusterlet   3/3     3            3           30s
```

Reference: https://github.com/open-cluster-management/registration-operator/blob/cb2abf6370e6b7e42c984ceb431f95aa69f3b796/Makefile#L132-L137

`$ kubectl config view --flatten --context=cluster1 --minify > ./config`

`$ kubectl create ns open-cluster-management-agent`

`$ kubectl create secret generic bootstrap-hub-kubeconfig --from-file=kubeconfig=./config -n open-cluster-management-agent`

`$ kubectl apply -f klusterlet-cluster1.yaml`

**NOTE:** Creates a ManagedCluster resource `$ kubectl get managedcluster`

Reference: https://github.com/open-cluster-management/registration-operator#what-is-next
```
$ kubectl get csr
NAME             AGE     SIGNERNAME                                    REQUESTOR              CONDITION
cluster1-lsxww   9m45s   kubernetes.io/kube-apiserver-client           minikube-user          Pending
csr-qsjm5        66m     kubernetes.io/kube-apiserver-client-kubelet   system:node:cluster1   Approved,Issued
```

`$ kubectl certificate approve cluster1-lsxww`

`$ kubectl patch managedcluster cluster1 -p='{"spec":{"hubAcceptsClient":true}}' --type=merge`

**ALERT:** Wait for deployments to be ready before proceeding
```
kubectl get deployments -n open-cluster-management-agent
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
klusterlet-registration-agent   3/3     3            3           3m59s
klusterlet-work-agent           3/3     3            3           3m59s
```

## Second OCM managed cluster

`$ minikube start --profile=cluster2`

**NOTE:** minikube automatically sets the default cluster to the new cluster, i.e cluster2. Hence some commands carry the explicit `--context=cluster1` when required.

### Repeat "Installing OLM"
`$ kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${olm_release}/crds.yaml`

`$ kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/${olm_release}/olm.yaml`

**ALERT:** Watch the deployments in "olm" namespace

### Repeat "Create operator group for OCM"
`$ kubectl create ns open-cluster-management`

`$ kubectl apply -f operatorgroup.yaml`

### Repeat "First managed cluster" with small tweaks!
`$ kubectl apply -f subscription-klusterlet.yml`

**ALERT:** Watch the deployments in "open-cluster-management" namespace

`$ kubectl create ns open-cluster-management-agent`

`$ kubectl create secret generic bootstrap-hub-kubeconfig --from-file=kubeconfig=./config -n open-cluster-management-agent`

`$ kubectl apply -f klusterlet-cluster2.yaml`

`$ kubectl get csr --context=cluster1`

`$ kubectl certificate approve cluster2-xxxx --context=cluster1`

`$ kubectl patch managedcluster cluster1 -p='{"spec":{"hubAcceptsClient":true}}' --type=merge --context=cluster1`

**ALERT:** Watch the deployments in "open-cluster-management-agent" namespace to ensure they are fine

## CRDs on the 2 clusters at the end of this!

```
$ kubectl get crd --context=cluster1 | grep -i manage
appliedmanifestworks.work.open-cluster-management.io           2021-02-11T01:33:06Z
clustermanagers.operator.open-cluster-management.io            2021-02-11T01:27:15Z
klusterlets.operator.open-cluster-management.io                2021-02-11T01:32:11Z
managedclusters.cluster.open-cluster-management.io             2021-02-11T01:28:51Z
managedclustersetbindings.cluster.open-cluster-management.io   2021-02-11T01:28:51Z
managedclustersets.cluster.open-cluster-management.io          2021-02-11T01:28:51Z
manifestworks.work.open-cluster-management.io                  2021-02-11T01:28:51Z
```

```
$ kubectl get crd --context=cluster2 | grep -i manage
appliedmanifestworks.work.open-cluster-management.io   2021-02-11T01:41:49Z
klusterlets.operator.open-cluster-management.io        2021-02-11T01:39:22Z
```

## Test an subscription/application deployment

**TODO!!!**
