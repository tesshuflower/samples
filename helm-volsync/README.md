Deploy helm app on managed clusters using the helm subscription.
The Policy attached creates the channel, subs, policy for all volsync addons.

When an addon is created or modified, all the volsync addon controller should do is to add an anntation to the ManagedCluster resource in this form : volsync=0.10.0 , where 0.10.0 is the defaul channel or the channel set by the user with the annotation. 
The Policy is going to take care of the subscription/placement creation, see below. The helm will be deployed on the ManagedCluster because the placement matches the cluster, based on the volsync=0.10.0  label.


Steps, followed by the policy:
1. creates the Channel resource ( the repo where the channel points needs only an index.yaml file containing the helm descriptions - this is all that this repo requires )

Sample for the index.yaml expected by the helm subscription : https://github.com/fxiang1/app-samples/blob/main/index.yaml

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: Channel
metadata:
  name: volsync-channel
  namespace: volsync-ns
spec:
  pathname: 'https://raw.githubusercontent.com/volsync/main' # this is where the index.xml is located
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
  name: mortgage
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

