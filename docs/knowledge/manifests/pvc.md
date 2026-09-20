# Manifest Reference — PersistentVolumeClaim

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Purpose
Application request for persistent storage.

## apiVersion / kind
- apiVersion: v1
- kind: PersistentVolumeClaim

## spec core
- accessModes
- resources.requests.storage
- storageClassName
- volumeMode
- volumeName
- selector
- dataSource / dataSourceRef

## Access modes
- ReadWriteOnce
- ReadOnlyMany
- ReadWriteMany
- ReadWriteOncePod

## Related resources
- PersistentVolume
- StorageClass
- CSI driver
- Pod volume/volumeMount

## Runtime concepts
- Pending/Bound/Lost
- dynamic provisioning
- binding mode
- expansion
- reclaim policy belongs to PV/StorageClass side

## Troubleshooting
```bash
kubectl get pvc,pv,storageclass
kubectl describe pvc <name>
kubectl get events --sort-by=.lastTimestamp
```

## Sources
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/storage-classes/
