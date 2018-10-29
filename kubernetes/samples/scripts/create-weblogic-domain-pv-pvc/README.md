# Creating Sample Persistent Volume and Persistent Volume Claim

This sample script demonstrates the creation of a Kubernetes persistent volume (PV) and persistent volume claim (PVC). The generated PV and PVC can then be used in a domain custom resource for the domain home or logs.

## Prerequisites

The following pre-requisites must be handled prior to running this script:
* The kubernetes namespace must already be created for the persistent volume claim unless the intention is to use the default namespace
* The host directory that will be used as the persistent volume must already exist and have the appropriate file permissions set.


## Using the script to create the Kubernetes resources

Run the create script, pointing it at an inputs file and output directory:

```
  ./create-pv-pvc.sh \
  -i create-pv-pvc-inputs.yaml \
  -o /path/to/output-directory
```

By default, the script generates two yaml files in `/path/to/output-directory`, namely `weblogic-sample-pv.yaml` and `weblogic-sample-pvc.yaml`. These two yaml files care be used to create the Kubernetes resources using `kubectl create -f` or `kubectl apply -f`.

```
  kubectl create -f weblogic-sample-pv.yaml
  kubectl create -f weblogic-sample-pvc.yaml

```

As a convenience, the script can optionally create the PV and PVC resources as well using the `-e` option.

If you copy the sample scripts to a different location, make sure that you copy everything in the `<weblogic-kubernetes-operator-project>/kubernetes/samples/scripts` directory togather into the target directory, maintaining the orignal directory heirachy.


## Configuration Parameters 

The following parameters in the inputs file can be customized if needed.

| Parameter | Definition | Default |
| --- | --- | --- |
| `domainUID` | ID of the domain resource that the generated PV and PVC will be dedicated to. Leave it empty if the PV and PVC are going to be shared by multiple domains. | no default |
| `namespace` | Kubernetes namespace to create the pvc. | `default` |
| `baseName` | Base name of the PV and PVC. The generated PV and PVC will be <baseName>-pv and <baseName>-pvc respectively | `weblogic-sample` |
| `weblogicDomainStoragePath` | Physical path of the storage for the persistent volume. | no default |
| `weblogicDomainStorageReclaimPolicy` | Kubernetes persistent volume reclaim policy for the persistent storage. The valid values are: 'Retain', 'Delete', and 'Recycle' | `Retain` |
| `weblogicDomainStorageSize` | Total storage allocated for the PVC. | `10Gi` |
| `weblogicDomainStorageType` | Type of storage. Legal values are `NFS` and `HOST_PATH`. If using 'NFS', weblogicDomainStorageNFSServer must be specified | `HOST_PATH` |
| `weblogicDomainStorageNFSServer`| Name of the IP address of the NFS server. This setting only applies if weblogicDomainStorateType is NFS  | no default |

By default the domainUID is left empty in the inputs file so that the generated PV and PVC can be shared by multiple domain resources. For use cases where per domain PV and PVC are desired, setting the domainUID will cause the generated PV and PVC associated with the domainUID specified.In the per domain PV and PVC case, the names of the generated yaml files and the Kubernetes PV and PVC objects are all decorated with the `domainUID`, and the PV and PVC obects are also labeled with the `domainUID`.

## Common problems

This section provides details of common problems that occur while running the script  and how to resolve them.

### Persistent volume provider not configured correctly

Possibly the most common problem experienced during testing was incorrect configuration of the persistent volume provider.  The persistent volume must be accessible to all Kubernetes nodes, and must be able to be mounted as Read/Write/Many.  If this is not the case, the domain creation will fail.

The simplest case is where the `HOST_PATH` provider is used.  This can be either with one Kubernetes node, or with the `HOST_PATH` residing in shared storage available at the same location on every node (for example, on an NFS mount).  In this case, the path used for the persistent volume must have its permission bits set to 777.

## Verifying the results 

The script will verify that the domain was created, and will report failure if there was any error.  However, it may be desirable to manually verify the domain, even if just to gain familiarity with the various Kubernetes objects that were created by the script.

### YAML files created by the script

The `create-pv-pvc.sh` script creates two yaml files and put them in the `/path/to/output-directory` directory.

The following are the contents of the yaml files with the default settings.

Here is the content of the `weblogic-sample-pvc.yaml`.

```
# Copyright 2018, Oracle Corporation and/or its affiliates.  All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: weblogic-sample-pvc
  namespace: default
  labels:
    weblogic.resourceVersion: domain-v1

  storageClassName: weblogic-sample-storage-class
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi 
```

Here is the content of the `weblogic-sample-pv.yaml`.
```
# Copyright 2018, Oracle Corporation and/or its affiliates.  All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.

apiVersion: v1
kind: PersistentVolume
metadata:
  name: weblogic-sample-pv
  labels:
    weblogic.resourceVersion: domain-v1
    # weblogic.domainUID: 
spec: 
  storageClassName: weblogic-sample-storage-class
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  # Valid values are Retain, Delete or Recycle
  persistentVolumeReclaimPolicy: Retain
  hostPath:
  # nfs:
    # server: %WEBLOGIC_DOMAIN_STORAGE_NFS_SERVER%
    path: "/scratch/k8s_dir"

```

### Verify the PV and PVC resources

To confirm that the domain custom resource was created, use this command:

```
kubectl describe pv
kubectl describe pvc -n NAMESPACE
```

Replace `NAMESPACE` with the namespace that the PVC was created in.  The output of this command will provide details of the domain, as shown in this example:

```
$ kubectl describe pv weblogic-sample-pv
Name:            weblogic-sample-pv
Labels:          weblogic.resourceVersion=domain-v1
Annotations:     pv.kubernetes.io/bound-by-controller=yes
StorageClass:    weblogic-sample-storage-class
Status:          Bound
Claim:           default/weblogic-sample-pvc
Reclaim Policy:  Retain
Access Modes:    RWX
Capacity:        10Gi
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /scratch/k8s_dir
    HostPathType:  
Events:            <none>

```

```
$ kubectl describe pvc weblogic-sample-pvc 
Name:          weblogic-sample-pvc
Namespace:     default
StorageClass:  weblogic-sample-storage-class
Status:        Bound
Volume:        weblogic-sample-pv
Labels:        weblogic.resourceVersion=domain-v1
Annotations:   pv.kubernetes.io/bind-completed=yes
               pv.kubernetes.io/bound-by-controller=yes
Finalizers:    []
Capacity:      10Gi
Access Modes:  RWX
Events:        <none>

```
