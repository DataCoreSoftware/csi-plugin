# Installing the DataCore CSI Plugin

## Prerequisites
In order to use the CSI plugin, you must have the following in place already:
  * DataCore SANsymphony 10.0 PSP7 Update 2 or later
  * DataCore REST Server 1.08 or later
  * A Kubernetes cluster with 1 or more master with the following feature gates enabled:
    * KubeletPluginsWatcher
    * CSINodeInfo
    * CSIDriverRegistry
* Each worker node that will need DataCore volumes must have the iSCSI initiator already installed, configured and connected to the DataCore storage servers that will be provisioning persistent storage volumes.

## Preparation
On each node in the cluster, ensure that an iSCSI connection has been established with the DataCore SDS servers. In the example below, an iSCSI connection is being established with a DataCore SDS iSCSI target at IP address 100.0.0.1
```
# iscsiadm -m discovery -t sendtargets -p 100.0.0.1 —-op new
# iscsiadm -m node —-login
```

Repeat this step for each iSCSI target you wish to connect to.

## Install 
Download the installer on your master node
```
wget https://demos.datacore.com/downloads/csi/csi.tar.gz
```

Untar the installer
```
tar -xzvf csi.tar.gz
```

Create the installation manifest
```
{
 "IScsiTargetPortals" : [ "IPAddress1", "IPAddress2", ... ], 
 "Username" : "Username for DataCore SDS",
 "Password" : "Password for DataCore SDS User",
 "RestServer" : "IPAddress of DataCore REST Server",
 "DataCoreArrayName" : "IPAddress/FQDN of one of the servers in the DataCore Server Group",
 "DataCoreImage" : "datacoresoftware/csi-plugin:1.0.1",
 "DeployAsPod" : false,
 "Namespace" : "kube-system",
 "VirtualDiskTemplate" : "Name of template to use in SANsymphony. Must already be created."
}
```

IScsiTargetPortals
  * An array of IP addresses of the target portals you wish each node of kubernetes cluster to be able to connect to. They must be reachable by iSCSI from each node. Each node must have already been connected to the SDS using iscsiadm.

Username
  * The username to use to connect to the SSY server group. It must be a member of the administrative role in SANsymphony.
  
Password
  * The password to for the user account for the SSY server group.

RestServer
  * The name or IP address of the DataCore REST server that will be used to communicate with the SANsymphony server group.
  
DataCoreArrayName
  * One of the servers in the server group that you want the Kubernetes nodes to connect to.
  
DeployAsPod
  * Should be set to false for clusters that have single or multiple masters. If you want to try the plugin on a single node you can set this too true.
  
Namespace
  * The Kubernetes namespace to deploy to. 
  
VirtualDiskTemplate
  * The name of the template to use to create persistent volumes from. This template must have been set up ahead of time prior to installation.

Run the installer pointing it to the file that was created above. Note this must be done with sudo or root access. 
```
# ./dcscsi -installcsi=datacoreinstall.json
2018-12-17T10:04:34-05:00 |INFO| starting installer configfile=datacoreinstall.json
2018-12-17T10:04:34-05:00 |INFO| reading config file
2018-12-17T10:04:34-05:00 |INFO|   TargetPortal=100.0.0.1
2018-12-17T10:04:34-05:00 |INFO|   TargetPortal=100.0.0.2
2018-12-17T10:04:34-05:00 |INFO|   Username=administrator
2018-12-17T10:04:34-05:00 |INFO|   Password=RGF0YWNvcmUx
2018-12-17T10:04:34-05:00 |INFO|   RestServer=100.0.0.1
2018-12-17T10:04:34-05:00 |INFO|   StorageArray=100.0.0.5
2018-12-17T10:04:34-05:00 |INFO|   DriverImage=datacoresoftware/csi-plugin:1.0.1
2018-12-17T10:04:34-05:00 |INFO|   DeployAsPod=false
2018-12-17T10:04:34-05:00 |INFO|   Namespace=kube-system
2018-12-17T10:04:34-05:00 |INFO| preparing installation directory
2018-12-17T10:04:34-05:00 |INFO| initializing orchestrator client
2018-12-17T10:04:35-05:00 |INFO| kubernetes found kubernetes client version=v1.12.3 kubernetes server version=v1.12.3
2018-12-17T10:04:35-05:00 |INFO| preparing installation manifests
2018-12-17T10:04:35-05:00 |INFO| checking array to ensure this host is registered storageserver=docker-ssy-bk1
2018-12-17T10:04:35-05:00 |INFO| did not find this host in the array - registering it
2018-12-17T10:04:35-05:00 |INFO| successfully logged in iqn=iqn.2000-08.com.datacore:docker-ssy-bk2-1 tp=100.0.0.1:3260
2018-12-17T10:04:35-05:00 |INFO| successfully logged in iqn=iqn.2000-08.com.datacore:docker-ssy-bk2-2 tp=100.0.0.2:3260
2018-12-17T10:04:35-05:00 |INFO| host is registered in storage array nodeid=4fb54e406edc47babb58770ed5421cdf storagearray=100.0.0.5
2018-12-17T10:04:35-05:00 |INFO| creating secrets
2018-12-17T10:04:36-05:00 |INFO| creating configmap
2018-12-17T10:04:36-05:00 |INFO| creating storageclass
2018-12-17T10:04:36-05:00 |INFO| creating cdp storageclass
2018-12-17T10:04:36-05:00 |INFO| creating rbac
2018-12-17T10:04:37-05:00 |INFO| creating attacher
2018-12-17T10:04:37-05:00 |INFO| creating provisioner
2018-12-17T10:04:37-05:00 |INFO| creating node
```

Ensure that the installation is completed successfully:
```
$ kubectl get statefulsets -n kube-system
NAME                              DESIRED   CURRENT   AGE
csi-attacher-datacore-plugin      1         1         2m13s
csi-provisioner-datacore-plugin   1         1         2m13s

$ kubectl get daemonsets -n kube-system
NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
csi-datacore-plugin       1         1         1       1            1           <none>                            2m39s
...
...

$ kubectl get storageclasses
NAME           PROVISIONER            AGE
datacore       com.datacore.csi.dvp   3m31s
datacore-cdp   com.datacore.csi.dvp   3m31s
```
