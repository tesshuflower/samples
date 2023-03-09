# ACM Persistent data backup and restore

Stateful application DR using ACM policies 
------

- [Scenario](#scenario)
- [Prerequisites](#prerequisites)
  - [Apply policies on the hub](#apply-policies-on-the-hub)
    - [Backup PolicySet](#backup-policyset)
    - [Restore PolicySet](#restore-policyset)
  - [Setup hdr-app-configmap ConfigMap](#setup-hdr-app-configmap-configmap)
  - [Install policy](#install-policy)
- [Backup applications](#backup-applications)
  - [Backup pre and post hooks](#backup-pre-and-post-hooks)
- [Restore applications](#restore-applications)
  - [Restore pre and post hooks](#restore-pre-and-post-hooks)
- [Testing Scenario - pacman](#testing-scenario)
- [Example of pre and post backup hooks](#example-of-pre-and-post-backup-hooks)
- [Usage considerations](#usage-considerations)

------

## Scenario
The Policies available here provide backup and restore support for stateful applications running on  managed clusters or hub. Velero is used to backup and restore applications data. The product is installed using the OADP operator, which the `oadp-hdr-app-install` policy installs and configure on each target cluster.

You can use these policies to backup stateful applications (policies under `oadp-hdr-app-backup-set` policySet) or to restore applications backups (policies under `oadp-hdr-app-restore-set` policySet).

The PolicySet is used to place the backup or restore policies on managed clusters.

The policies should be installed on the hub managing clusters where you want to create stateful applications backups, or the hub managing clusters where you plan to restore the application backups. 

Both backup and restore policies can be installed on the same hub, if this hub manages clusters where applications need to be backed up or restored. 

A managed cluster can be a backup target if the PlacementRule for the `oadp-hdr-app-backup-set` PolicySet includes this managed cluster.

A managed cluster can be a restore target if the PlacementRule for the `oadp-hdr-app-restore-set` PolicySet includes this managed cluster. 

A managed cluster can be both a backup and restore target, if the PlacementRule from the backup and restore PolicySets matches this cluster.


## Prerequisites


### Setup hdr-app-configmap ConfigMap


Before you install the backup and restore Policies and PolicySets on the hub, you have to update the `hdr-app-configmap` ConfigMap values. 

You create the configmap on the hub, the same hub where the policies will be installed.

The configmap sets configuration options for the backup storage location, for the backup schedule backing up applications, and for the restore resource used to restore applications backups.

Make sure you <b>update all settings with valid values</b> before applying the `hdr-app-configmap` resource on the hub.

<b>Note</b>:

- The `dpa.spec` property defines the storage location properties. The default value shows the `dpa.spec` format for using an S3 bucket. Update this to match the type of storage location you want to use.
- You can still upate the `hdr-app-configmap` properties after the `ConfigMap` was applied to the hub. When you do that, the backup settings on the managed clusters where the PolicySet has been applied will be automatically updated with the new values for the `hdr-app-configmap`.  For example, to restore a new backup, update the `restore.backupName` property on the `hdr-app-configmap` on the hub; the change is pushed to the deployed policies on the managed cluster and a restore resource matching the new properties will be created there.
- To create multiple backup configurations you want to deploy the `PolicySet` on separate namespaces on the hub. For example, if you want to have a backup for `pacman` on `cluster1` and a backup for `pacman` and `busybox` on `cluster2`, you can create a namespace for the two options and apply the `ConfigMap` and policies in both ( before applying the `ConfigMap` on each namespace, it should be updated to match the options for that deployment ):

```
oc create ns pacman-policy
oc project pacman-policy
oc apply -k ./policy
```

```
oc create ns pac-bbox-policy
oc project pac-bbox-policy
oc apply -k ./policy
```

### Prereq for placing this policies on the hub

<b>Important:</b>

If the hub is one of the clusters where this policy will be placed, and the `backupNS=open-cluster-management-backup` then first enable cluster-backup on `MultiClusterHub`. 

The MultiClusterHub resource looks for the cluster-backup option and if set to false, it uninstalls OADP from the `open-cluster-management-backup` and deletes the namespace.



### Apply policies on the hub

Run `oc apply -k ./policy` to apply all resources at the same time on the hub. 

Use the `kustomization.yaml` available with this project to install the backup and restore Policies and PolicySets on the hub. 
None of the resources available with the project have a namespace setup. This is to allow the user to apply the Policies in different configurations for a different set of managed clusters.

#### Backup PolicySet

For example, if you need to backup `pacman` application running on `managed-1`, `managed-2` clusters and want to backup `mysql` on `managed-3`, you can apply the backup policy set and use 2 different namespaces:

1. For the `pacman` app:
- update the  `hdr-app-configmap` and set `backup.nsToBackup: "[\"pacman-ns\"]"`
- update the `acm-app-backup-placement` PlacementRule under `acm-app-backup-policy-set.yaml` to match the `managed-1`, `managed-2` clusters (or add the `acm-pv-dr=backup` label to the clusters).
- create `dr-app-pacman-ns` ( or use your own preferred ns ) - where the backup policy for pacman app will be created on the hub
- create the policies under that ns 

`oc project dr-app-pacman-ns`
`oc kustomize ./policy`


1. For the `mysql` app:
- update the  `hdr-app-configmap` and set `backup.nsToBackup: "[\"mysql-ns\"]"`
- update the `acm-app-backup-placement` PlacementRule under `acm-app-backup-policy-set.yaml` to match the `managed-3` cluster (or add the `acm-pv-dr=backup` label to the cluster).
- create `dr-app-mysql-ns` ( or use your own preferred ns, with no backup policy applied ) - where the backup policy for mysql app will be created on the hub
- create the policies under that ns 

`oc project dr-app-mysql-ns`
`oc kustomize ./policy`



#### Restore PolicySet

If you need to restore `pacman` application running on `managed-4` cluster and want to restore `mysql` on `managed-5`, you can apply the restore policy set and use 2 different namespaces:

1. For the `pacman` app:
- update the  `hdr-app-configmap` and set `restore.nsToRestore: "[\"pacman-ns\"]`
- update `restore.backupName` and set the backup name you want to restore
- update the `acm-app-restore-placement` PlacementRule under `acm-app-restore-policy-set.yaml` to match the `managed-4` cluster (or add the `acm-pv-dr=restore` label to the cluster).
- create `dr-app-pacman-ns` ( or use your own preferred ns ) - where the restore policy for pacman app will be created on the hub
- create the policies under that ns 

`oc project dr-app-pacman-ns`
`oc kustomize ./policy`


1. For the `mysql` app:
- update the  `hdr-app-configmap` and set `restore.nsToRestore: "[\"mysql-ns\"]`
- update `restore.backupName` and set the backup name you want to restore
- update the `acm-app-restore-placement` PlacementRule under `acm-app-restore-policy-set.yaml` to match the `managed-5` cluster (or add the `acm-pv-dr=restore` label to the cluster).
- create `dr-app-mysql-ns` ( or use your own preferred ns, with no restore policy applied ) - where the restore policy for mysql app will be created on the hub
- create the policies under that ns 

`oc project dr-app-mysql-ns`
`oc kustomize ./policy`



### Install policy 


The `oadp-hdr-app-install` policy is used to install OADP and configure the connection to the storage location.

This policy is created on the hub managing clusters where you want to create stateful applications backups, or where you restore these backup.  
The policy is set to enforce.

Make sure the `hdr-app-configmap`'s storage settings are properly set before applying this policy.

The  `oadp-hdr-app-install` installs velero and configures the connection to the storage. It also informs on any runtime or configuration error.


## Backup applications

If the hub manages clusters where stateful applications are running, and you want to create backups for these applications, then on the hub you must use the `oadp-hdr-app-backup-set` policySet.

If the managed cluster (or hub) matches the PlacementRule for the `oadp-hdr-app-backup-set` policySet, then the oadp-hdr-app-backup policy is propagated to this cluster for an application backup schedule and the cluster produces application backups. Read [Backup PolicySet](#backup-policyset) for more details on how this works.


Make sure the `hdr-app-configmap`'s backup schedule resource settings are properly set before applying this policy.

This policy is enforced by default.

The  policy also informs on any backup configuration error.

This policy creates a velero schedule to all managed clusters that match the PlacementRule for the `oadp-hdr-app-backup-set` policySet.

The schedule is used to backup applications resources and PVs. The name of the schedule is `acm-pv-<pv-storage-region>-<cls-name>`
The schedule uses the `backup.nsToBackup` `hdr-app-configmap` property to specify the namespaces for the applications to backup. 

### Backup pre and post hooks 

If you want to prepare the application before running the backup, you can use the following annotations on a pod to make [Velero execute a backup hook](https://velero.io/docs/v1.9/backup-hooks/) when backing up the pod:

`pre.hook.backup.velero.io/container`
`pre.hook.backup.velero.io/command`


## Restore applications


If the hub manages clusters where stateful applications backups must be restored, then you must install the `oadp-hdr-app-backup-set` policySet.

If the managed cluster (or hub) matches the PlacementRule for the `oadp-hdr-app-restore-set` policySet, then the oadp-hdr-app-restore policy is propagated to this cluster for a restore operation. Read [Restore PolicySet](#restore-policyset) for more details on how this works.

This cluster restores applications backup.

Make sure the `hdr-app-configmap`'s restore resource settings are properly set before applying this policy.

This policy is enforced by default.

The  policy also informs on any restore configuration error.

This policy creates a velero restore resource to all managed clusters that match the PlacementRule for the `oadp-hdr-app-restore-set` policySet.
The restore resource is used to restore applications resources and PVs
from a selected backup.
The restore uses the `nsToRestore` hdr-app-configmap property to specify the namespaces for the applications to restore.

### Restore pre and post hooks 

If you want to run some commands pre and post restore, you can use the following annotations on a pod to make [Velero execute a restore hook](https://velero.io/docs/v1.9/restore-hooks/) when restoring the pod:


`init.hook.restore.velero.io/container-image`
The container image for the init container to be added.
`init.hook.restore.velero.io/container-name`
The name for the init container that is being added.
`init.hook.restore.velero.io/command`
This is the ENTRYPOINT for the init container being added. This command is not executed within a shell and the container image’s ENTRYPOINT is used if this is not provided.


<b>Note:</b>
1. The restore operation doesn't update PV or PVC resources if the restored resource is already on the cluster where the app data is restored. Make sure your application is not installed on that cluster before the restore operation is executed - the PV or PVC available with the restore app should not exist on the restore cluster prior to the restore operation.
2. The restore cluster must be able to access the region where the restore PV and snapshots are located.
3. The restore cluster must have a `VolumeSnapshotLocation` velero resource with the same name as the one used by the backup `volumeSnapshotLocations` property. The `VolumeSnapshotLocation` resource from the restore cluster must point to the backed up PV snapshots location, otherwise the restore operation will fail to restore the PV.



## Testing scenario

Use the pacman app to test the policies. (You can use 2 separate hubs for the sample below, each managing one cluster. Place the pacman app on the hub managing c1)

1. On the hub, have 2 managed clusters c1 and c1.
2. On the hub, create the pacman application subscription 
- create an app in the `pacman-ns` namespace, of type git and point to this app https://github.com/tesshuflower/demo/tree/main/pacman
- place this app on c1
- play the app, create some users and save the data.
- verify that you can see the saved data when you launch the pacman again.
3. On the hub, install the polices above, using the instructions from the readme. See [Backup PolicySet](#backup-policyset) and [Restore PolicySet](#restore-policyset)


Backup step:<br>

5. On the hub, set the `backup.nsToBackup: "[\"pacman-ns\"]" ` on the `hdr-app-configmap` resource. This will backup all resources from the `pacman-ns`
6. Place the install and backup policies on c1 : create this label on c1 `acm-pv-dr=backup`


Restore step:<br>

7. On the hub, on the `hdr-app-configmap` resource:
- set the `restore.nsToRestore: "[\"pacman-ns\"]" `. This will restore all resources from the `pacman-ns`
- set the `restore.backupName:` and use a backup name created from step 6
8. Place the restore policy on c2 : create this label on c2 `acm-pv-dr=restore`
9. You should see the pacman app on c2; launch the pacman app and verify that you see the data saved when running the app on c1.


# Example of pre and post backup hooks:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: mongo
  name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      name: mongo
  template:
    metadata:
      labels:
        name: mongo
      annotations:
        pre.hook.backup.velero.io/container: mongo
        pre.hook.backup.velero.io/command: '["/sbin/fsfreeze", "--freeze", "/bitnami/mongodb/data/db"]'
        post.hook.backup.velero.io/container: mongo
        post.hook.backup.velero.io/command: '["/sbin/fsfreeze", "--unfreeze", "/bitnami/mongodb/data/db"]'
    spec:
      containers:
      - image: bitnami/mongodb:5.0.14
        name: mongo
        ports:
        - name: mongo
          containerPort: 27017
        volumeMounts:
          - name: mongo-db
            mountPath: /bitnami/mongodb/data/db
      volumes:
        - name: mongo-db
          persistentVolumeClaim:
            claimName: mongo-storage
```

# Usage considerations

1. When restoring a backup make sure the restore cluster doesn't already contain PVs and PVClaims with the same name as the ones restored with the backup. The PV and PVClaims will not be updated if they already exist on the restore cluster. 
2. Using restic for backing up PVs (use the configmap from the `./policy/input/restic` folder)
    - Use this option if the PV snapshot is not supported your backup and restore clusters are running on different platforms or they are in different regions and can't share PV snapshots, otherwise you can use the PV Snapshot approach.
    - See restic limitations here https://velero.io/docs/v1.9/restic/#limitations 
3. Uing PV Snapshot backup (use the configmap from the  `./policy/input/pv-snap` folder)
    - PVStorage for the backup resource must match the location of the PVs to be backed up. 
    - You cannot backup PVs from different regions/locations in the same backup, since a backup points to only one PVStorage
    -  You cannot restore a PV unless the restore resource points to the same PVStorage as the backup; so the restore cluster must have access to the PV snapshot storage location.
    - PV backup is storage/platform specific; you need the same storage class usage on both source ( where you backup the PV and take the snapshots) and on target cluster ( where you restore the PV snapshot )
4. Policy template adds new data if the CRD allows: Updating the config map and reapplying it on hub could end up in duplicating resource properties. For example, if I want to update the `snapshotLocations` location property to `us-est-1`, after I update the config map I end up with a DataProtectionApplication object containing two `snapshotLocations`, one for the old value and another for the new one.  The fix would be to :
    -  delete the policy and reapply - but this removes all the resources created by the policy, so a bit too aggressive.
    - manually remove the old property - hard to do and error prone if the resource was placed on a lot of clusters