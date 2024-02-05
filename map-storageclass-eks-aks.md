## Map the storage class of your EKS to AKS

As AKS and EKS use different storageClassNames for the persistent volume claims, we need to create a configMap that will translate the source storageClassNames to target class name. Since EKS use `gp2` as a default storage class for EBS volumes so we will create the file configMap.yaml with the following content to translate it to `default` class for your AKS cluster.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  # any name can be used; Velero uses the labels (below)
  # to identify it rather than the name
  name: change-storage-class-config
  # must be in the velero namespace
  namespace: velero
  # the below labels should be used verbatim in your
  # ConfigMap.
  labels:
    # this value-less label identifies the ConfigMap as
    # config for a plugin (i.e. the built-in restore item action plugin)
    velero.io/plugin-config: ""
    # this label identifies the name and kind of plugin
    # that this ConfigMap is for.
    velero.io/change-storage-class: RestoreItemAction
data:
  # add 1+ key-value pairs here, where the key is the old
  # storage class name and the value is the new storage
  # class name.
  gp2: default
```
Apply the configMap to your AKS cluster with the following command.

```bash
kubectl apply -f configMap.yaml -n velero
```

Another way is to have the same StorageClass available in AKS as used by EKS Persistent Volumes. For example, if the Storage Class of the PVs is `gp2`, create the same on AKS using below template:

```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
provisioner: disk.csi.azure.com
parameters:
  skuName: StandardSSD_LRS
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Next Step
[Restore the backup to AKS cluster](restore-aks-cluster.md)