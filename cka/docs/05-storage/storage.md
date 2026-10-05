# Storage

Up: [CKA hub](../../CKA_2026_Complete_Crash_Course.md) · Domain 5 — Storage (10%) · Prev: [Helm and Kustomize](../04-workloads-scheduling/helm-and-kustomize.md)

## Objects

- **PersistentVolume (PV)**: a piece of storage in the cluster, created by an admin or dynamically.
- **PersistentVolumeClaim (PVC)**: a request for storage with size and access mode.
- **StorageClass**: a template for dynamic provisioning through a CSI driver.

A PVC binds to a PV that matches its size, access mode and storage class. Dynamic provisioning creates the PV when the PVC appears.

## Access modes

| Mode | Meaning |
|---|---|
| ReadWriteOnce (RWO) | One node mounts read-write |
| ReadOnlyMany (ROX) | Many nodes mount read-only |
| ReadWriteMany (RWX) | Many nodes mount read-write (needs a storage backend that supports it) |
| ReadWriteOncePod (RWOP) | One pod mounts read-write |

## Reclaim policy

- `Retain`: keep the PV and data after the PVC is deleted; manual cleanup.
- `Delete`: delete the PV and the backing storage with the PVC.

## Create a PV and PVC (static)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity: {storage: 1Gi}
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests: {storage: 500Mi}
```

`hostPath` is for labs only. Real clusters use a CSI-backed StorageClass.

## Use a PVC in a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-demo
```

## Dynamic provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: <csi-driver-name>      # from the cluster
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

`WaitForFirstConsumer` delays binding until a pod uses the PVC, so the volume lands in the same zone as the pod.

## Expand a PVC

Requires `allowVolumeExpansion: true` on the StorageClass.

```bash
kubectl patch pvc pvc-demo -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl get pvc pvc-demo
```

## Troubleshooting

| Symptom | Check |
|---|---|
| PVC stuck Pending | `describe pvc` events: no matching PV, no StorageClass, provisioner error |
| Pod stuck Pending with volume | PVC not Bound; access mode incompatible with the node |
| Expansion does nothing | `allowVolumeExpansion` false, or the filesystem resize has not run yet (restart the pod) |
| Data gone after delete | Reclaim policy was Delete |

```bash
kubectl get pv,pvc -A
kubectl describe pvc <name>
kubectl get storageclass
```

## Common mistakes

- Setting `storageClassName` to a class that does not exist.
- Asking for RWX on a backend that only supports RWO.
- Forgetting `allowVolumeExpansion` before trying to resize.

## Quick reference

```bash
kubectl get pv,pvc,sc -A
kubectl describe pvc <name>
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"<size>"}}}}'
```
