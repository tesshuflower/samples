This sample shows how to deploy volsync on managed clusters  using the helm subscription.
The Policy attached here [volsync-addon-subscription](./policy/create-subscription-policy.yaml) creates the channel, subscription, placement for all volsync addons.
It uses the [volsync-acm-config.yaml](./policy/volsync-acm-config.yaml) ConfigMap to define the volsync chart channel repo and other volsync default settings ( stable channel, etc.

The Policy creates the main volsync channel and for each volsync chart version, create the subscription and placement. The volsync chart will be deployed on all managed clusters with a label volsync that matches the subscription placement( for example volsync=0.10.0  label.) 

What the volsync addon controller should do:
- When a volsync addon is created or modified, the volsync addon controller should add a label annotation to the ManagedCluster resource in this form : volsync=0.10.0 , where 0.10.0 is the default channel or the channel set using the with `operator-subscription-channel` annotation. 
- When a volsync addon is deleted, remove the  volsync label from the ManagedCluster resource. This will remove the volsync Subscription on the managed cluster and uninstall the resources.


Steps, followed by the policy:
1. The Policy creates the Channel resource ( the repo where the channel points needs only an index.yaml file containing the volsync chart descriptions - this is all that this repo needs to provide )

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Channel
metadata:
  name: volsync-channel
  namespace: volsync-ns
spec:
  pathname: 'https://backube.github.io/helm-charts'
  type: HelmRepo
```

2. When a volsync addon is created, the policy creates a Subscription + Placement rule using either the default volsync channel or the channel set with the operator-subscription-channel annotation. All addons pointing to the same channel will share the same hub Subscription and Placement.
The policy creates one subscription + placement for each volsync channel.

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Subscription
metadata:
  name: volsync-subscription-0.10.0
  namespace: volsync
spec:
  channel: >-
    volsync-ns/volsync-channel
  name: volsync
  packageFilter:
    version: 0.10.0
  packageOverrides:
    - packageAlias: volsync
      packageName: volsync
  placement:
    placementRef:
      kind: Placement
      name: volsync-placement-0.10.0
```

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: volsync-placement-0.10.0
  namespace: volsync
spec:
  clusterSets:
    - global
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: volsync
              operator: In
              values:
                - 0.10.0
```

