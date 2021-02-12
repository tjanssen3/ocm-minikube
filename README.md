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
Reference: https://github.com/kubernetes/cluster-registry

`$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/cluster-registry/master/cluster-registry-crd.yaml`

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

`$ kubectl apply -f subscription-multicluster-operators.yaml`

**ALERT:** Watch the deployments in "open-cluster-management-agent" namespace to ensure they are fine
```
watch kubectl get deployments -n open-cluster-management
NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
cluster-manager                                  3/3     3            3           17m
multicluster-operators-application               1/1     1            1           91s
multicluster-operators-hub-subscription          1/1     1            1           91s
multicluster-operators-standalone-subscription   1/1     1            1           91s
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

`$ kubectl patch managedcluster cluster2 -p='{"spec":{"hubAcceptsClient":true}}' --type=merge --context=cluster1`

**ALERT:** Watch the deployments in "open-cluster-management-agent" namespace to ensure they are fine

`$ kubectl apply -f subscription-multicluster-operators.yaml`

**ALERT:** Watch the deployments in "open-cluster-management-agent" namespace to ensure they are fine

`$ kubectl config use-context cluster1`

`$ kubectl patch managedcluster cluster1 -p='{"metadata":{"labels":{"usage":"test"}}}' --type=merge`

`$ kubectl patch managedcluster cluster2 -p='{"metadata":{"labels":{"usage":"development"}}}' --type=merge`

## CRDs on the 2 clusters at the end of this!

```
$ kubectl get crds | grep -i management
appliedmanifestworks.work.open-cluster-management.io           2021-02-12T19:15:37Z
channels.apps.open-cluster-management.io                       2021-02-12T19:07:34Z
clustermanagers.operator.open-cluster-management.io            2021-02-12T18:51:28Z
deployables.apps.open-cluster-management.io                    2021-02-12T19:07:34Z
helmreleases.apps.open-cluster-management.io                   2021-02-12T19:07:34Z
klusterlets.operator.open-cluster-management.io                2021-02-12T19:10:10Z
managedclusters.cluster.open-cluster-management.io             2021-02-12T18:52:17Z
managedclustersetbindings.cluster.open-cluster-management.io   2021-02-12T18:52:17Z
managedclustersets.cluster.open-cluster-management.io          2021-02-12T18:52:17Z
manifestworks.work.open-cluster-management.io                  2021-02-12T18:52:17Z
placementrules.apps.open-cluster-management.io                 2021-02-12T19:07:34Z
subscriptions.apps.open-cluster-management.io                  2021-02-12T19:07:34Z
```

```
$ kubectl get crds --context=cluster2 | grep -i management
appliedmanifestworks.work.open-cluster-management.io   2021-02-12T19:25:12Z
channels.apps.open-cluster-management.io               2021-02-12T19:22:10Z
deployables.apps.open-cluster-management.io            2021-02-12T19:22:10Z
helmreleases.apps.open-cluster-management.io           2021-02-12T19:22:10Z
klusterlets.operator.open-cluster-management.io        2021-02-12T19:23:47Z
placementrules.apps.open-cluster-management.io         2021-02-12T19:22:10Z
subscriptions.apps.open-cluster-management.io          2021-02-12T19:22:10Z
```

## Test an subscription/application deployment

`$ git clone git@github.com:open-cluster-management/application-samples.git`

`$ cd application-samples`

`$ kubectl apply -k subscriptions/channel`

**NOTE:** Need some edits to the Subscription manifest to get the right github branch/path, apply the following diff,
```
diff --git a/subscriptions/book-import/subscription.yaml b/subscriptions/book-import/subscription.yaml
index 69fcb6f..affcc9c 100644
--- a/subscriptions/book-import/subscription.yaml
+++ b/subscriptions/book-import/subscription.yaml
@@ -3,8 +3,8 @@ apiVersion: apps.open-cluster-management.io/v1
 kind: Subscription
 metadata:
   annotations:
-    apps.open-cluster-management.io/git-branch: master
-    apps.open-cluster-management.io/git-path: book-import/app
+    apps.open-cluster-management.io/github-branch: main
+    apps.open-cluster-management.io/github-path: book-import
   labels:
     app: book-import
   name: book-import
```

`$ kubectl apply -k subscriptions/book-import`

**TODO**: The above creates "deployables" in the application namespace, but there is no real deployment on the managed cluster (still hunting for errors that is causing this). IOW, the Subscription deployable remains in the Propogated state and does not become Active

```
$ kubectl get deployable -n book-import
NAME                                             TEMPLATE-KIND   TEMPLATE-APIVERSION                  AGE   STATUS
book-import-book-import-book-import-deployment   Deployment      apps/v1                              8s    
book-import-book-import-book-import-route        Route           route.openshift.io/v1                8s    
book-import-book-import-book-import-service      Service         v1                                   8s    
book-import-deployable                           Subscription    apps.open-cluster-management.io/v1   8s    Propagated
```
