Kubernetes Snapshotting Proposal
================================

**Authors:** [Cindy Wang](https://github.com/ciwang), [Jing Xu](https://github.com/jinxu97), [Tomas Smetana](https://github.com/tsmetana)

## Background

Many storage systems (GCE PD, Amazon EBS, etc.) provide the ability to create "snapshots" of a persistent volumes to protect against data loss. Snapshots can be used in place of a traditional backup system to back up and restore primary and critical data. Snapshots allow for quick data backup (for example, it takes a fraction of a second to create a GCE PD snapshot) and offer fast recovery time objectives (RTOs) and recovery point objectives (RPOs).

Typical existing backup solutions offer on demand or scheduled snapshots.

An application developer using a storage may want to create a snapshot before an update or other major event. Kubernetes does not currently offer a standardized snapshot API for creating, listing, deleting, and restoring snapshots on an arbitrary volume.

Existing solutions for scheduled snapshotting include [cron jobs](https://forums.aws.amazon.com/message.jspa?messageID=570265) and [external storage drivers](http://rancher.com/introducing-convoy-a-docker-volume-driver-for-backup-and-recovery-of-persistent-data/). Some cloud storage volumes can be configured to take automatic snapshots, but this is specified on the volumes themselves.

## Objectives

For the first version of snapshotting support in Kubernetes, only on-demand snapshots will be supported. Features listed in the roadmap for future versions are also nongoals.

* Goal 1: Enable *on-demand* snapshots of Kubernetes persistent volumes by application developers.

    * Nongoal: Enable *automatic* periodic snapshotting for direct volumes in pods.

* Goal 2: Expose standardized snapshotting operations to create, list and delete snapshots in Kubernetes REST API.

* Goal 3: Implement snapshotting interface for GCE PDs.

    * Stretch goal: Implement snapshotting interface for any non GCE PD volumes.

### Feature Roadmap

Major features, planned for the first version:

* On demand snapshots

    * API to create new snapshots

    * API to list snapshots available to the user

    * API to delete existing snapshots

    * API to create a persistent volume from a snapshot

#### Future Features

The features that are not planned for the first version of the API bus should be considered in future versions:

* Creating snapshots

    * Scheduled and periodic snapshots

    * Aplication initiated on-demand snapshot creation

    * Support snapshot per PVC, pod or StatefulSet

    * Support snapshots for non-cloud storage volumes (i.e. plugins that require actions to be triggered from the node)

    * Support application-consistent snapshots (coordinate distributed snapshots across multiple volumes)

    * Enable to create a pod/statefulsets with snapshots

* List snapshots

    * Enable to get the list of all snapshots for a specified persistent volume

    * Enable to get the list of all snapshots for a pod/StatefulSet

* Delete snapshots

    * Enable to automatic garbage collect older snapshots when storage is limited

* Quota management

    * Enable to set quota for limiting how many snapshots could be taken and saved

    * When quota is exceeded, delete the oldest snapshots automatically

## Requirements

### Performance

* Time SLA from issuing a snapshot to completion:

* The period we are interested is the time between the scheduled snapshot time and the time the snapshot is finishes uploading to its storage location.

* This should be on the order of a few minutes.

### Reliability

* Data corruption

    * Though it is generally recommended to stop application writes before executing the snapshot command, we will not do this automatically for several reasons:

        * GCE and Amazon can create snapshots while the application is running.

        * Stopping application writes cannot be done from the master and varies by application, so doing so will introduce unnecessary complexity and permission issues in the code.

        * Most file systems and server applications are (and should be) able to restore inconsistent snapshots the same way as a disk that underwent an unclean shutdown.

    * The data consistency would be best-effort only: e.g., call fsfreeze prior the snapshot on filesystems that support it.

* Snapshot failure

    * Case: Failure during external process, such as during API call or upload

        * Log error, retry until success (indefinitely)

    * Case: Failure within Kubernetes, such as controller restarts

        *  If the master restarts in the middle of a snapshot operation, then the controller will find a snapshot request in pending state and should be able to successfully finish the operation.

## Solution Overview

There are a few uniqueness related to snapshots:

* Both users and admins might create snapshots. Users should only get access to the snapshots belong to their namespaces. For this aspect, snapshot objects should be in user namespace. Admins might to choose expose the snapshots they created to some users who have access to those volumes.

* After snapshots are taken, users might use them to create new volumes or restore the existing volumes back the time when the snapshot is taken.

* There are use cases that data from snapshots taken from one namespace need to be accessible by users in another namespace.

* For security purpose, if a snapshot object is created by a user, kubernetes should prevent other users duplicating this object in a different namespace if they happen to get the snapshot name.

* There might be some existing snapshots taken by admins/users and they want to use those snapshots through kubernetes API interface.

* **Create:**

    1. The user creates a `Snapshot` refrencing a persistent volume claim bound to a persistent volume

    2. The controller fulfils the `Snapshot` by creating a snapshot using the volume plugins.

* **List:**

    1. The user is able to list all the `Snapshot`s in the namespace

* **Delete:**

    1. The user deletes the `Snapshot`

    2. The controller removes the on-disk snapshot

* **Promote snapshot to PV:**

    1. The user creates a persistent volume claim referencing the snapshot object

    2. The controller will use the snapshot object and create a persistent volume from it

    3. The PVC is bound to the newly created PV containing the data from the snapshot


A new controller responsible solely for snapshot operations will be added to the controllermanager on the master. This controller will watch the API server for new `Snapshot` objects. When a new `Snapshot` is created, it will trigger the appropriate snapshot creation logic for the underlying persistent volume type.

The snapshot operation is a no-op for volume plugins that do not support snapshots via an API call (i.e. non-cloud storage).

### API


* The `Snapshot` object

```
// Snapshot is a user's request for a snapshot of a persistent volume claim
type Snapshot struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Spec defines the snapshot
	// +optional
	Spec SnapshotSpec

	// Status represents the current information about a snapshot
	// +optional
	Status SnapshotStatus
}

// SnapshotSpec describes the attributes of a snapshot
type SnapshotSpec struct {
    // The PVC being snapshotted
    PersistentVolumeClaim snapshottedVolumeClaim
}
```

```yaml
apiVersion: v1
kind: Snapshot
metadata:
  name: my-snap-1
  namespace: user-ns-1
spec:
  persistentVolumeClaim: my-snapshotted-pvc
  status:
    phase: bound
```

* PVC control loop in master

    * If a new `Snapshot` of a PVC, search for PV of volume type that implements SnapshottableVolumePlugin. If one is available, use it, update the API object with status `pending`. Otherwise, reject the claim and post an event to the `Snapshot`, set its status to `failed`. When the on-disk snapshot operation finishes, update the API objects status to `ready` and set its metadata timestamp.

### Create Snapshot Logic

To create a snapshot:

* Acquire operation lock for volume so that no other attach or detach operations can be started for volume.

    * Abort if there is already a pending operation for the specified volume (main loop will retry, if needed).

* Spawn a new thread:

    * Execute the volume-specific logic to create a snapshot of the persistent volume referenced by the PVC.

    * For any errors, log the error, and terminate the thread (the main controller will retry as needed).

    * Once a snapshot is created successfully:

        * Make a call to the API server to add the new snapshot ID/timestamp to the `Snapshot` API object, update its
        status.

### Workflow Schema

![Snapshot workflow diagram](volume-snapshots-workflow.png "Snapshot workflow diagram")

## Use Cases

### Alice wants to backup her MySQL database data

Alice is a DB admin who runs a MySQL database and needs to backup the data on a remote server prior to the database
upgrade. She has a short maintenance window dedicated to the operation that allows her to pause the dabase only for
a short while. Alice will therefore stop the database, create a snapshot of the data, re-start the database and after
that start time-consuming network transfer to the backup server.

The database is running in a pod with the data stored on a persistent volume:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    name: mysql
spec:
  containers:
    - resources:
        limits :
          cpu: 0.5
      image: openshift/mysql-55-centos7
      name: mysql
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpassword
        - name: MYSQL_USER
          value: wp_user
        - name: MYSQL_PASSWORD
          value: wp_pass
        - name: MYSQL_DATABASE
          value: wp_db
      ports:
        - containerPort: 3306
          name: mysql
      volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql/data
  volumes:
    - name: mysql-persistent-storage
      persistentVolumeClaim:
      claimName: claim-mysql
```

The persistent volume is bound to the `claim-mysql` PVC which needs to be snapshotted. Since Alice has some downtime
allowed she may lock the database tables for a moment to ensure the backup would be consistent:
```
mysql> FLUSH TABLES WITH READ LOCK;
```
Now she is ready to create a snapshot of the `claim-mysql` PVC. She creates `snap.yaml` file:

```yaml
apiVersion: v1
kind: Snapshot
metadata:
  name: mysql-snapshot
  namespace: default
spec:
  persistentVolumeClaim: claim-mysql
```

```
$ kubectl create -f snap.yaml
```

This will result in a new snapshot being created by the controller. Alice would wait until the snapshot is complete:
```
$ kubectl get snapshots

NAME             STATUS
mysql-snapshot   ready
```
Now it's OK to unlock the database tables and the database may return to normal operation:
```
mysql> UNLOCK TABLES;
```
Alice can now get to the snapshotted data and start syncing them to the remote server. First she needs to promote the
snapshot to a PV by creating a new PVC:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: snapshot-data-claim
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  snapshotRequest: mysql-snapshot
```
Once the claim is bound to a persistent volume Alice creates a job to sync the data with a remote backup server:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mysql-sync
spec:
  template:
    metadata:
      name: mysql-sync
    spec:
      containers:
      - name: mysql-sync
        image: rsync
        command: "rsync -av /mnt/data alice@backup.example.com:mysql_backups"
      restartPolicy: Never
      volumeMounts:
        - name: snapshot-data
          mountPath: /mnt/data
  volumes:
    - name: snapshot-data
      persistentVolumeClaim:
      claimName: snapshot-data-claim
```

Alice will wait for the job to finish and then may delete both the `snapshot-data-claim` PVC as well as `mysql-snapshot`
request (which will delete also the snapshot object):
```
$ kubectl delete pvc snapshot-data-claim
$ kubectl delete snapshot mysql-snapshot
```
<!---
Open questions:

* What has more value: scheduled snapshotting or exposing snapshotting/backups as a standardized API?

    * It seems that the API route is a bit more feasible in implementation and can also be fully utilized.

        * Can the API call methods on VolumePlugins? Yeah via controller

    * The scheduler gives users functionality that doesn't already exist, but required adding an entirely new controller

* Should the list and restore operations be part of v1?

* Do we call them snapshots or backups?

    * From the SIG email: "The snapshot should not be suggested to be a backup in any documentation, because in practice is is necessary, but not sufficient, when conducting a backup of a stateful application."

* At what minimum granularity should snapshots be allowed?

* How do we store information about the most recent snapshot in case the controller restarts?

* In case of error, do we err on the side of fewer or more snapshots?

Snapshot Scheduler

1. PVC API Object

A new field, backupSchedule, will be added to the PVC API Object. The value of this field must be a cron expression.

* CRUD operations on snapshot schedules

    * Create: Specify a snapshot within a PVC spec as a [cron expression](http://crontab-generator.org/)

        * The cron expression provides flexibility to decrease the interval between snapshots in future versions

    * Read: Display snapshot schedule to user via kubectl get pvc

    * Update: Do not support changing the snapshot schedule for an existing PVC

    * Delete: Do not support deleting the snapshot schedule for an existing PVC

        * In v1, the snapshot schedule is tied to the lifecycle of the PVC. Update and delete operations are therefore not supported. In future versions, this may be done using kubectl edit pvc/name

* Validation

    * Cron expressions must have a 0 in the minutes place and use exact, not interval syntax

        * [EBS](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/TakeScheduledSnapshot.html) appears to be able to take snapshots at the granularity of minutes, GCE PD takes at most minutes. Therefore for v1, we ensure that snapshots are taken at most hourly and at exact times (rather than at time intervals).

    * If Kubernetes cannot find a PV that supports snapshotting via its API, reject the PVC and display an error message to the user

 Objective

Goal: Enable automatic periodic snapshotting (NOTE:  A snapshot is a read-only copy of a disk.)  for all kubernetes volume plugins.

Goal: Implement snapshotting interface for GCE PDs.

Goal: Protect against data loss by allowing users to restore snapshots of their disks.

Nongoal: Implement snapshotting support on Kubernetes for non GCE PD volumes.

Nongoal: Use snapshotting to provide additional features such as migration.

 Background

Many storage systems (GCE PD, Amazon EBS, NFS, etc.) provide the ability to create "snapshots" of a persistent volumes to protect against data loss. Snapshots can be used in place of a traditional backup system to back up and restore primary and critical data. Snapshots allow for quick data backup (for example, it takes a fraction of a second to create a GCE PD snapshot) and offer fast recovery time objectives (RTOs) and recovery point objectives (RPOs).

Currently, no container orchestration software (i.e. Kubernetes and its competitors) provide snapshot scheduling for application storage.

Existing solutions for automatic snapshotting include [cron jobs](https://forums.aws.amazon.com/message.jspa?messageID=570265)/shell scripts. Some volumes can be configured to take automatic snapshots, but this is specified on the volumes themselves, not via their associated applications. Snapshotting support gives Kubernetes clear competitive advantage for users who want automatic snapshotting on their volumes, and particularly those who want to configure application-specific schedules.

 what is the value case? Who wants this? What do we enable by implementing this?

I think it introduces a lot of complexity, so what is the pay off? That should be clear in the document.  Do mesos, or swarm or our competition implement this? AWS? Just curious.

Requirements

Functionality

Should this support PVs, direct volumes, or both?

Should we support deletion?

Should we support restores?

Automated schedule -- times or intervals? Before major event?

Performance

Snapshots are supposed to provide timely state freezing. What is the SLA from issuing one to it completing?

* GCE: The snapshot operation takes [a fraction of a second](https://cloudplatform.googleblog.com/2013/10/persistent-disk-backups-using-snapshots.html). If file writes can be paused, they should be paused until the snapshot is created (but can be restarted while it is pending). If file writes cannot be paused, the volume should be unmounted before snapshotting then remounted afterwards.

    * Pending = uploading to GCE

* EBS is the same, but if the volume is the root device the instance should be stopped before snapshotting

Reliability

How do we ascertain that deletions happen when we want them to?

For the same reasons that Kubernetes should not expose a direct create-snapshot command, it should also not allow users to delete snapshots for arbitrary volumes from Kubernetes.

We may, however, want to allow users to set a snapshotExpiryPeriod and delete snapshots once they have reached certain age. At this point we do not see an immediate need to implement automatic deletion (re:Saad) but may want to revisit this.

What happens when the snapshot fails as these are async operations?

Retry (for some time period? indefinitely?) and log the error

Other

What is the UI for seeing the list of snapshots?

In the case of GCE PD, the snapshots are uploaded to cloud storage. They are visible and manageable from the GCE console. The same applies for other cloud storage providers (i.e. Amazon). Otherwise, users may need to ssh into the device and access a ./snapshot or similar directory. In other words, users will continue to access snapshots in the same way as they have been while creating manual snapshots.

Overview

There are several design options for the design of each layer of implementation as follows.

1. **Public API:**

Users will specify a snapshotting schedule for particular volumes, which Kubernetes will then execute automatically. There are several options for where this specification can happen. In order from most to least invasive:

    1. New Volume API object

        1. Currently, pods, PVs, and PVCs are API objects, but Volume is not. A volume is represented as a field within pod/PV objects and its details are lost upon destruction of its enclosing object.

        2. We define Volume to be a brand new API object, with a snapshot schedule attribute that specifies the time at which Kubernetes should call out to the volume plugin to create a snapshot.

        3. The Volume API object will be referenced by the pod/PV API objects. The new Volume object exists entirely independently of the Pod object.

        4. Pros

            1. Snapshot schedule conflicts: Since a single Volume API object ideally refers to a single volume, each volume has a single unique snapshot schedule. In the case where the same underlying PD is used by different pods which specify different snapshot schedules, we have a straightforward way of identifying and resolving the conflicts. Instead of using extra space to create duplicate snapshots, we can decide to, for example, use the most frequent snapshot schedule.

        5. Cons

            2. Heavyweight codewise; involves changing and touching a lot of existing code.

            3. Potentially bad UX: How is the Volume API object created?

                1. By the user independently of the pod (i.e. with something like my-volume.yaml). In order to create 1 pod with a volume, the user needs to create 2 yaml files and run 2 commands.

                2. When a unique volume is specified in a pod or PV spec.

    2. Directly in volume definition in the pod/PV object

        6. When specifying a volume as part of the pod or PV spec, users have the option to include an extra attribute, e.g. ssTimes, to denote the snapshot schedule.

        7. Pros

            4. Easy for users to implement and understand

        8. Cons

            5. The same underlying PD may be used by different pods. In this case, we need to resolve when and how often to take snapshots. If two pods specify the same snapshot time for the same PD, we should not perform two snapshots at that time. However, there is no unique global identifier for a volume defined in a pod definition--its identifying details are particular to the volume plugin used.

            6. Replica sets have the same pod spec and support needs to be added so that underlying volume used does not create new snapshots for each member of the set.

    3. Only in PV object

        9. When specifying a volume as part of the PV spec, users have the option to include an extra attribute, e.g. ssTimes, to denote the snapshot schedule.

        10. Pros

            7. Slightly cleaner than (b). It logically makes more sense to specify snapshotting at the time of the persistent volume definition (as opposed to in the pod definition) since the snapshot schedule is a volume property.

        11. Cons

            8. No support for direct volumes

            9. Only useful for PVs that do not already have automatic snapshotting tools (e.g. Schedule Snapshot Wizard for iSCSI) -- many do and the same can be achieved with a simple cron job

            10. Same problems as (b) with respect to non-unique resources. We may have 2 PV API objects for the same underlying disk and need to resolve conflicting/duplicated schedules.

    4. Annotations: key value pairs on API object

        12. User experience is the same as (b)

        13. Instead of storing the snapshot attribute on the pod/PV API object, save this information in an annotation. For instance, if we define a pod with two volumes we might have {"ssTimes-vol1": [1,5], “ssTimes-vol2”: [2,17]} where the values are slices of integer values representing UTC hours.

        14. Pros

            11. Less invasive to the codebase than (a-c)

        15. Cons

            12. Same problems as (b-c) with non-unique resources. The only difference here is the API object representation.

2. **Business logic:**

    5. Does this go on the master, node, or both?

        16. Where the snapshot is stored

            13. GCE, Amazon: cloud storage

            14. Others stored on volume itself (gluster) or external drive (iSCSI)

        17. Requirements for snapshot operation

            15. Application flush, sync, and fsfreeze before creating snapshot

    6. Suggestion:

        18. New SnapshotController on master

            16. Controller keeps a list of active pods/volumes, schedule for each, last snapshot

            17. If controller restarts and we miss a snapshot in the process, just skip it

                3. Alternatively, try creating the snapshot up to the time + retryPeriod (see 5)

            18. If snapshotting call fails, retry for an amount of time specified in retryPeriod

            19. Timekeeping mechanism: something similar to [cron](http://stackoverflow.com/questions/3982957/how-does-cron-internally-schedule-jobs); keep list of snapshot times, calculate time until next snapshot, and sleep for that period

        19. Logic to prepare the disk for snapshotting on node

            20. Application I/Os need to be flushed and the filesystem should be frozen before snapshotting (on GCE PD)

    7. Alternatives: login entirely on node

        20. Problems:

            21. If pod moves from one node to another

                4. A different node is in now in charge of snapshotting

                5. If the volume plugin requires external memory for snapshots, we need to move the existing data

            22. If the same pod exists on two different nodes, which node is in charge

3. **Volume plugin interface/internal API:**

    8. Allow VolumePlugins to implement the SnapshottableVolumePlugin interface (structure similar to AttachableVolumePlugin)

    9. When logic is triggered for a snapshot by the SnapshotController, the SnapshottableVolumePlugin calls out to volume plugin API to create snapshot

    10. Similar to volume.attach call

4. **Other questions:**

    11. Snapshot period

    12. Time or period

    13. What is our SLO around time accuracy?

        21. Best effort, but no guarantees (depends on time or period) -- if going with time.

    14. What if we miss a snapshot?

        22. We will retry (assuming this means that we failed) -- take at the nearest next opportunity

    15. Will we know when an operation has failed?  How do we report that?

        23. Get response from volume plugin API, log in kubelet log, generate Kube event in success and failure cases

    16. Will we be responsible for GCing old snapshots?

        24. Maybe this can be explicit non-goal, in the future can automate garbage collection

    17. If the pod dies do we continue creating snapshots?

    18. How to communicate errors (PD doesn't support snapshotting, time period unsupported)

    19. Off schedule snapshotting like before an application upgrade

    20. We may want to take snapshots of encrypted disks. For instance, for GCE PDs, the encryption key must be passed to gcloud to snapshot an encrypted disk. Should Kubernetes handle this?

Options, pros, cons, suggestion/recommendation

Example 1b

During pod creation, a user can specify a pod definition in a yaml file. As part of this specification, users should be able to denote a [list of] times at which an existing snapshot command can be executed on the pod's associated volume.

For a simple example, take the definition of a [pod using a GCE PD](http://kubernetes.io/docs/user-guide/volumes/#example-pod-2):

apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    # This GCE PD must already exist.
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4

Introduce a new field into the volume spec:

apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    # This GCE PD must already exist.
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4

**    ssTimes: ****[1, 5]**

 Caveats

* Snapshotting should not be exposed to the user through the Kubernetes API (via an operation such as create-snapshot) because

    * this does not provide value to the user and only adds an extra layer of indirection/complexity.

    * ?

 Dependencies

* Kubernetes

* Persistent volume snapshot support through API

    * POST https://www.googleapis.com/compute/v1/projects/example-project/zones/us-central1-f/disks/example-disk/createSnapshot



--->
