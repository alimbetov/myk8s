# Walkthrough — File worker + PVC

## Архитектура

```text
file-worker Pod
    |
volumeMount /data/work
    |
volume name=work
    |
PVC work-data
    |
PV / CSI storage
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [PVC reference](../../docs/knowledge/manifests/pvc.md)

## Mapping chain

```text
volumeMount.name
      =
volumes.name

persistentVolumeClaim.claimName
      =
PVC.metadata.name
```

## RWOP choice

`ReadWriteOncePod` означает один Pod writer cluster-wide.

Поэтому example использует:

```yaml
replicas: 1
```

Если сделать 3 replicas, storage semantics конфликтуют с workload scaling.

## Когда PVC оправдан

- application требует filesystem API;
- large local processing;
- legacy component;
- state tied to filesystem semantics.

Если нужен object model, S3 может быть лучше.

## Pod replacement

```text
Pod deleted
 -> PVC remains
 -> replacement mounts same PVC
 -> files remain
```

## Failure scenarios

- no default StorageClass -> PVC Pending;
- unsupported RWOP -> provisioning issue;
- second replica -> mount conflict;
- disk full -> app fails writes;
- PVC missing -> Pod Pending.

## Проверка

```bash
kubectl get pvc,pv,storageclass
kubectl describe pvc work-data
kubectl describe pod <pod>
```

## Главное

Persistent volume сохраняет data across Pod replacement, но не является backup.
