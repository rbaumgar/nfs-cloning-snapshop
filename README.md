# Cloning and Snapshop NFS Volume Examples

This example is based on the k8s default NFS class: https://github.com/kubernetes-csi/csi-driver-nfs.

## Install CSI driver for NFS

```shell
git clone https://github.com/kubernetes-csi/csi-driver-nfs.git
cd csi-driver-nfs
./deploy/install-driver.sh v4.7.0 local
```

## Create a storageclass

```shell
kubectl apply -f examples/storageclass-nfs.yaml
```

Check csidriver

```shell
kubectl get csidriver  nfs.csi.k8s.io -o json | jq .spec
{
  "attachRequired": false,
  "fsGroupPolicy": "File",
  "podInfoOnMount": false,
  "requiresRepublish": false,
  "seLinuxMount": false,
  "storageCapacity": false,
  "volumeLifecycleModes": [
    "Persistent"
  ]
}
```

## Patch the storageProfile

When default cloneStrategy = snapshot following error may occur:

`persistentvolumeclaims "tmp-pvc-3d603a8c-ea6e-4fb7-9a01-de5420735f93" is forbidden: only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize`

```shell
kubectl patch storageProfile nfs-csi-storage --type=json -p='[ { "op": "add", "path": "/spec", 
         "value": { "cloneStrategy": "csi-clone" } } ]'
```

## Create a project

```shell
kubectl new-project backup
```

## Create a PVC

```shell
kubectl apply -f examples/pvc-nfs.yaml
persistentvolumeclaim/pvc-nfs created
```

## Create a pod

```shell
kubectl apply -f examples/pod-nginx-nfs.yaml
pod/nginx-nfs created
```

## Check the log

```shell
kubectl exec nginx-nfs -- ls /mnt/nfs
outfile

kubectl exec nginx-nfs -- cat /mnt/nfs/outfile
Fri May 6 10:27:24 UTC 2024
Fri May 6 10:27:25 UTC 2024
Fri May 6 10:27:26 UTC 2024
```

## Create a PVC from an existing PVC
Make sure the application is not writing data to source NFS share.

```shell
kubectl apply -f examples/pvc-nfs-cloning.yaml
persistentvolumeclaim/pvc-nfs-cloning created
```

## Check cloned PVC

```shell
kubectl describe pvc pvc-nfs-cloning
Name:          pvc-nfs-cloning
Namespace:     backup
StorageClass:  nfs-csi-storage
Status:        Bound
Volume:        pvc-67c5f925-8286-40fc-9712-41e3a970dd3e
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: nfs.csi.k8s.io
               volume.kubernetes.io/storage-provisioner: nfs.csi.k8s.io
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      10Gi
Access Modes:  RWX
VolumeMode:    Filesystem
DataSource:
  Kind:   PersistentVolumeClaim
  Name:   pvc-nfs
Used By:  <none>
Events:
  Type    Reason                 Age                From                                                          Message
  ----    ------                 ----               ----                                                          -------
  Normal  ExternalProvisioning   40s (x2 over 40s)  persistentvolume-controller                                   Waiting for a volume to be created either by the external provisioner 'nfs.csi.k8s.io' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
  Normal  Provisioning           40s                nfs.csi.k8s.io_master-1_e6cc3b52-55db-44d6-9252-57a7e2f25f27  External provisioner is provisioning volume for claim "backup/pvc-nfs-cloning"
  Normal  ProvisioningSucceeded  40s                nfs.csi.k8s.io_master-1_e6cc3b52-55db-44d6-9252-57a7e2f25f27  Successfully provisioned volume pvc-67c5f925-8286-40fc-9712-41e3a970dd3e
```

## Create a pod based on the cloned PVC

```shell
kubectl apply -f examples/pod-nginx-cloning.yaml
pod/nginx-nfs-cloning created
```

## Check the log that it is continuing after the snapshot

```shell
kubectl exec nginx-nfs-cloning -- ls /mnt/nfs
outfile

kubectl exec nginx-nfs-cloning -- cat /mnt/nfs/outfile
...
Mon May 6 10:39:16 UTC 2024
Mon May 6 10:39:17 UTC 2024
Mon May 6 10:39:18 UTC 2024
Mon May 6 10:54:41 UTC 2024
Mon May 6 10:54:42 UTC 2024
Mon May 6 10:54:43 UTC 2024
```

You will find the logs going until 10:39:18 UTC. And then the log starts again in the new pod. You can find the time in the metadata of the snapshot.

## Create a SnapshotClass

Snapshot is supported by the CSI NFS driver from v4.3.0.

```shell
kubectl apply -f examples/snapshotclass-nfs.yaml
volumesnapshotclass.snapshot.storage.k8s.io/nfs-csi-snapshot configured
```

## Create a VolumeSnapshot

```shell
kubectl apply -f examples/volumesnapshot-nfs.yaml
volumesnapshot.snapshot.storage.k8s.io/pvc-nfs-snapshot created
```

## Check the VolumeSnapshot

```shell
kubectl describe volumesnapshot pvc-nfs-snapshot
Name:         pvc-nfs-snapshot
Namespace:    backup
Labels:       <none>
Annotations:  <none>
API Version:  snapshot.storage.k8s.io/v1
Kind:         VolumeSnapshot
Metadata:
  Creation Timestamp:  2024-05-03T16:38:49Z
  Finalizers:
    snapshot.storage.kubernetes.io/volumesnapshot-as-source-protection
    snapshot.storage.kubernetes.io/volumesnapshot-bound-protection
  Generation:        1
  Resource Version:  2316859602
  UID:               08a5fb25-b11f-408e-a8e3-7ddb3f055628
Spec:
  Source:
    Persistent Volume Claim Name:  pvc-nfs
  Volume Snapshot Class Name:      nfs-csi-snapclass
Status:
  Bound Volume Snapshot Content Name:  snapcontent-08a5fb25-b11f-408e-a8e3-7ddb3f055628
  Ready To Use:                        true
Events:
  Type    Reason            Age   From                 Message
  ----    ------            ----  ----                 -------
  Normal  CreatingSnapshot  7s    snapshot-controller  Waiting for a snapshot backup/pvc-nfs-snapshot to be created by the CSI driver.
  Normal   SnapshotCreated         20s    snapshot-controller  Snapshot backup/pvc-nfs-snapshot was successfully created by the CSI driver.
  Normal   SnapshotReady           20s    snapshot-controller  Snapshot backup/pvc-nfs-snapshot is ready to use.

kubectl get volumesnapshot pvc-nfs-snapshot -o jsonpath={.status.readyToUse}
true
```

## Create a new PVC based on the snapshot

```shell
kubectl apply -f examples/pvc-nfs-snapshot-restored.yaml
persistentvolumeclaim/pvc-nfs-snapshot-restored created

kubectl get pvc pvc-nfs-snapshot-restored
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
pvc-nfs-snapshot-restored   Bound    pvc-793dbe69-0dc1-4c95-a141-886d63a947e4   10Gi       RWX            nfs-csi-storage   42s
```

## Create a pod based on the snapshot

```shell
kubectl apply -f examples/pod-nginx-snapshot-restored.yaml
pod/nginx-nfs-restored-snapshot created
```

## Check the log that it is continuing after the snapshot

```shell
kubectl exec nginx-nfs-restored-snapshot -- ls /mnt/nfs
outfile

kubectl exec nginx-nfs-restored-snapshot -- cat /mnt/nfs/outfile
...
Mon May 6 13:52:41 UTC 2024
Mon May 6 13:52:42 UTC 2024
Mon May 6 13:52:43 UTC 2024
Mon May 6 13:52:44 UTC 2024
Mon May 6 13:52:45 UTC 2024
Mon May 6 14:30:39 UTC 2024
Mon May 6 14:30:40 UTC 2024
Mon May 6 14:30:41 UTC 2024
```

You will find the logs going until 13:52:45 UTC. And then the log starts again in the new pod. You can find the time in the metadata of the snapshot.

```shell
kubectl get volumesnapshot pvc-nfs-snapshot -o jsonpath={.metadata.creationTimestamp}
2024-05-06T13:52:45Z
```

## Cleanup

```shell
kubectl delete project backup
project.project.openshift.io "backup" deleted
```