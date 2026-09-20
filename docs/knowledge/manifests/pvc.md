# Manifest Reference — PersistentVolumeClaim

## Учебная схема объекта

### Где PVC находится в storage chain

```text
Pod
 |
 | volumeMount
 v
Volume
 |
 | claimName
 v
PVC
 |
 v
PV
 |
 v
StorageClass / CSI
 |
 v
physical storage
```

### Annotated fragment

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: work-data

spec:
  accessModes:
    - ReadWriteOncePod   # strict single-Pod writer, if CSI supports

  resources:
    requests:
      storage: 10Gi
```

Pod:

```yaml
volumes:
  - name: work
    persistentVolumeClaim:
      claimName: work-data

containers:
  - volumeMounts:
      - name: work
        mountPath: /data/work
```

### Не путать

```text
PVC persistence
 !=
backup

ReadWriteOnce
 !=
strict single Pod
```

Практика: [PVC worker](../../../showcases/05-file-worker-pvc/annotated.yaml).

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Назначение

PVC — namespaced request на persistent storage.

```text
Pod
 -> PVC
 -> PV
 -> StorageClass / CSI
 -> physical storage
```

## 2. Minimal manifest

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## 3. Production-style example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: orders-data
spec:
  accessModes:
    - ReadWriteOncePod
  volumeMode: Filesystem
  storageClassName: fast-csi
  resources:
    requests:
      storage: 20Gi
```

Support зависит от CSI/storage platform.

## 4. accessModes

- `ReadWriteOnce` (RWO)
- `ReadOnlyMany` (ROX)
- `ReadWriteMany` (RWX)
- `ReadWriteOncePod` (RWOP)

## 5. ReadWriteOnce

Read-write mount single node.

Важно: multiple Pods на одном node могут использовать volume.

RWO != strict single-Pod writer.

## 6. ReadWriteOncePod

Stable с Kubernetes 1.29.

Strict single-Pod read-write mount cluster-wide.

Только CSI volumes с compatible implementation.

## 7. ReadOnlyMany

Read-only many nodes, если storage implementation поддерживает.

Access mode используется для matching/binding и не является универсальной application-level ACL.

## 8. ReadWriteMany

Read-write many nodes, если backend поддерживает.

Не все CSI drivers/storage systems поддерживают RWX.

## 9. resources.requests.storage

```yaml
resources:
  requests:
    storage: 20Gi
```

Requested capacity.

## 10. storageClassName

```yaml
storageClassName: fast-csi
```

Выбор StorageClass.

Если omitted и есть default StorageClass, default provisioning может примениться.

Empty string и omitted — разные semantics.

## 11. volumeMode

- `Filesystem`
- `Block`

**Default: Filesystem.**

Filesystem -> mount path.
Block -> raw device via volumeDevices.

## 12. volumeName

Explicit pre-binding к конкретному PV.

Обычно dynamic provisioning предпочтительнее; manual binding — special recovery/migration case.

## 13. selector

PVC может выбрать PV по labels.

Selector усложняет dynamic provisioning; использовать осознанно.

## 14. dataSource / dataSourceRef

Provision from snapshot/clone/custom source при наличии соответствующих APIs/controllers.

Advanced storage workflow.

## 15. Status phases

Common:
- Pending
- Bound
- Lost

Pending может быть как error, так и нормальным состоянием при delayed binding.

## 16. Dynamic provisioning

```text
PVC created
 -> StorageClass selected
 -> CSI provisioner
 -> backing storage
 -> PV
 -> PVC Bound
```

## 17. WaitForFirstConsumer

StorageClass может отложить binding/provisioning до появления Pod scheduling context, чтобы выбрать правильную topology/zone.

## 18. Pod mount

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: orders-data

containers:
  - volumeMounts:
      - name: data
        mountPath: /data
```

Pod и PVC в одном namespace.

## 19. Lifecycle vs Pod

```text
Pod deleted
 -> PVC remains
 -> replacement Pod may mount same claim
```

## 20. PVC deletion / reclaim

Physical storage lifecycle зависит от PV reclaim policy и provisioner.

Common:
- Delete
- Retain

До delete production claim проверить reclaim policy.

## 21. Expansion

Если StorageClass allows expansion и CSI поддерживает:
- storage request можно увеличить;
- shrink обычным PVC resize не поддерживается;
- filesystem resize timing зависит implementation.

## 22. StatefulSet

VolumeClaimTemplates создают per-Pod PVC.

```text
postgres-0 -> data-postgres-0
postgres-1 -> data-postgres-1
```

Data lifecycle должен проектироваться независимо от Pod lifecycle.

## 23. Failure — no StorageClass

```bash
kubectl get pvc
kubectl describe pvc orders-data
kubectl get storageclass
```

## 24. Failure — unsupported access mode

Requested RWX, backend supports only RWO -> provisioning/binding fails or claim Pending.

## 25. Failure — topology mismatch

Volume zone A, Pod constrained zone B -> scheduling/attach errors.

```bash
kubectl describe pod <pod>
kubectl describe pvc <pvc>
kubectl describe pv <pv>
```

## 26. Failure — disk full

PVC Bound != infinite space.

Restart Pod does not clear persistent data.

Нужны capacity metrics, cleanup/expand runbook.

## 27. Security

- fs permissions/fsGroup;
- storage encryption depends backend;
- snapshots/backups sensitive;
- reclaim policy/compliance;
- access mode not substitute for application authorization.

## 28. Backup distinction

PVC persistence != backup.

Не защищает от logical delete/corruption.

DB/queue requires application-consistent backup/PITR strategy.

## 29. Day-2

```bash
kubectl get pvc
kubectl get pv
kubectl get storageclass
kubectl describe pvc orders-data
kubectl describe pv <pv>
kubectl get events --sort-by=.lastTimestamp
```

## 30. Anti-patterns

- RWO interpreted as single Pod;
- delete claim without reclaim check;
- PVC considered backup;
- emptyDir for durable DB;
- RWX selected without application semantics;
- no capacity monitoring.

## 31. CKAD

Must know PVC create, accessModes, storage request, volumeMode basics, Pod mount, Pending/Bound diagnostics.

## Production-like examples

- [PVC mounted by file worker](../../../showcases/05-file-worker-pvc/README.md)
- [Per-Pod PVC through volumeClaimTemplates](../../../showcases/18-statefulset-headless-service/README.md)

Каждый showcase содержит `WALKTHROUGH.md` с разбором mapping'ов и `all.yaml` с полной композицией.

## Sources

- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/storage-classes/
- https://kubernetes.io/docs/tasks/administer-cluster/change-pv-access-mode-readwriteoncepod/
