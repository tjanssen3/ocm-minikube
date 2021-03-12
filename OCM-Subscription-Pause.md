# Ability to pause a Subscription on PlacementRule change when it has the added ramen label

# Code changes

  - Deployable change: https://github.com/ShyamsundarR/multicloud-operators-deployable/tree/pause-on-plchange
  - Subscription change: https://github.com/ShyamsundarR/multicloud-operators-subscription/tree/pause-on-plchange

# New label

- Subscription CR to contain added label, as below, to trigger pause functionality:
  ```
  metadata:
    labels:
      ramendr: protected
  ```
- Subscription will be paused if PlacementRule change triggres a reconcile of the subscription, or deployable. The Subscription will get an added label as follows,
  ```
  metadata:
    labels:
      ramendr: protected
      subscription-pause: "true"
  ```

To unpause subscription, edit the subscription and remove the label named `subscription-pause`

To trigger a placement rule change, edit the placement rule for the subscription to change the cluster label.
NOTE: Ideally the test would be to turn-off/disconnect current managedcluster on which Subscription is placed, for DR tests

## Build and deploy

- Clone above repositories, change to branch `pause-on-plchange`
- Build: TRAVIS_BUILD=0 make build-images
  - Builds local images named,
    - quay.io/open-cluster-management/multicluster-operators-deployable:canary
    - quay.io/open-cluster-management/multicluster-operators-subscription:canary
- Copy image to minikube cluster, or push to cluster of choice
- Change the following deployments on the HUB to pick the new image,
  - deployment: multicluster-operators-application namespace: multicluster-operators
    - Update image,
      - From: image: quay.io/open-cluster-management/multicluster-operators-deployable:community-latest
      - To: image: quay.io/open-cluster-management/multicluster-operators-deployable:canary
    - Update imagePullPolicy to "Never" for the container using the above image
  - deployment: multicluster-operators-subscription namespace: multicluster-operators
    - Update image,
      - From: image: quay.io/open-cluster-management/multicluster-operators-subscription:community-latest
      - To: image: quay.io/open-cluster-management/multicluster-operators-subscription:canary
    - Update imagePullPolicy to "Never" for the container using the above image
- Restart the deployments (although the change should trigger the same)

# Gotchas

- Subscription does not start paused, as it seems not triggered by a PlacementRule
- Shutting down the managed cluster for the subscription does move the Subscription to another available managed cluster, but takes time to detect this
- Restarting the managed cluster that was down, does not seem to be removing the workload on the cluster
  - Need to retest without the changes to pause to ensure core understanding is correct
