# Installing the DataCore CSI Plugin

## Prerequisites
In order to use the CSI plugin, 
* The minimum supported upgrade path is:
  * DataCore SANsymphony 10.0 PSP12 Update 2 or later
  * DataCore REST Support 2.05 or later
  * Kubernetes 1.18 or later
* Each worker node that requires DataCore volumes must have the iSCSI initiator already installed, configured, and connected to the DataCore storage servers that provision persistent storage volume.
* For VolumeSnapshot feature to work install the CRDs and Snapshot Controller,
  * git clone https://github.com/kubernetes-csi/external-snapshotter.git
  * kubectl apply -f external-snapshotter/client/config/crd/
  * kubectl apply -f  external-snapshotter/deploy/kubernetes/snapshot-controller

## Preparation
On each node in the cluster, ensure that an iSCSI connection has been established with the DataCore SDS servers. In the example below, an iSCSI connection is being established with a DataCore SDS iSCSI target at IP address 100.0.0.1
```
# iscsiadm -m discovery -t sendtargets -p 100.0.0.1 —-op new
# iscsiadm -m node —-login
```

Repeat this step for each iSCSI target you wish to connect to.

## What's new in this release
  * Support for snapshot
  * Support for creating PersistentVolumeClaim from snapshot 
  * Support for creating PersistentVolumeClaim from another PersistentVolumeClaim (clone)

## Limitations
  * Maximum name length of PersistentVolumeClaim (PVC) supported by Datacore CSI plugin is 64 bytes.
  * Maximum name length of VolumeSnapshot supported by DataCore CSI plugin is 64 bytes. 

## Install
Create the datacore namespace
```
kubectl create namespace datacore
```

Create a configuration map. Create a new yaml file and copy the following content to it:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: csi-ssy-config
  namespace: datacore
data:
  rest-server: {RESTSERVER}
  storage-server: {SSVSERVER}
  vd-template: {TEMPLATE}
```
Replace `{RESTSERVER}` with the IP address/FQDN of the DataCore REST server. 
Replace `{SSVSERVER}` with the IP address/FQDN of one of the servers in the DataCore server group.
Replace `{TEMPLATE}` with the name of the Virtual Disk template to use. This template must already be created. Note that it is also possible to create storage classes and specify a different virtual disk template than the one provided here.
Save this file and apply it with `kubectl`.

Create a secret with the credentials to use to connect to the SANsymphony server group. The credentials must be base64 encoded. Create a new yaml file and copy the following content to it:
```
apiVersion: v1
kind: Secret
metadata:
  name: csi-ssy-secret
  namespace: datacore
data:
  username: {USERNAME}
  password: {PASSWORD}
```
Replace `{USERNAME}` with the base64 encoded value of the username to connect to the SANsymphony server group. For example, if the username is administrator, then `{USERNAME}` would be replaced by the output of `echo -n administrator | base64`.
Replace `{PASSWORD}` with the password for the account used to connect to the SANsymphony server group. For example, if the password is `password`, then `{PASSWORD}` would be replaced by the output of `echo -n password | base64`.
Save this file and apply it with `kubectl`.

Apply the installer manifest
`kubectl apply -f https://raw.githubusercontent.com/DataCoreSoftware/csi-plugin/master/datacore-csi-1.2.0.yaml`

Check that the plugin has been deployed.
`kubectl get pods -n datacore`
A successful deployment will have the controller and node pods running.

## Examples: Creating PersistentVolumeClaim
After following the above deployment, a new storage class called `disk.csi.ssy.datacore.com` would have been created. Using this storage class you can provision virtual disks with PVCs. The following example provisions a new 100Gi virtual disk named `dev-pvc`:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dev-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: disk.csi.ssy.datacore.com
```
## Examples: Creating VolumeSnapshot
The following example creates snapshot of dev-pvc.
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: dev-pvc-snap
spec:
  volumeSnapshotClassName: csi-ssy-datacore-snapclass
  source:
    persistentVolumeClaimName: dev-pvc
```
## Examples: Creating PersistentVolumeClaim from VolumeSnapshot
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-from-snap
spec:
  storageClassName: disk.csi.ssy.datacore.com
  dataSource:
    kind: VolumeSnapshot
    name: dev-pvc-snap
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```   
## Examples: Creating PersistentVolumeClaim from PersistentVolumeClaim
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-from-pvc
spec:
  storageClassName: disk.csi.ssy.datacore.com
  dataSource:
    kind: PersistentVolumeClaim
    name: dev-pvc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
