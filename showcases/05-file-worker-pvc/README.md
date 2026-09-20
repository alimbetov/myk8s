# Showcase 05 — File-processing worker with PVC

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Полный связный manifest](all.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
Deployment worker
   |
   +--> ConfigMap
   +--> Secret
   +--> PVC work-data
            |
            v
       persistent volume
```

## Когда это полезно

Пример демонстрирует сервис, которому действительно нужен filesystem interface:
- временный staging больших файлов;
- legacy library, работающая с files;
- controlled shared work directory при поддерживаемом storage access mode.

Если нужен обычный upload/download object model, S3-compatible object storage часто проще и масштабируемее.

## Mapping

PVC:

```yaml
metadata:
  name: work-data
```

Pod:

```yaml
volumes:
  - name: work
    persistentVolumeClaim:
      claimName: work-data
```

Container:

```yaml
volumeMounts:
  - name: work
    mountPath: /data/work
```

Цепочка:

```text
volumeMount.name
      =
volume.name
      ↓
persistentVolumeClaim.claimName
      =
PVC.metadata.name
```

## Access mode

В примере `ReadWriteOncePod`: strict single Pod writer. Это означает, что Deployment нельзя просто масштабировать до нескольких simultaneous Pods, ожидая общий RW storage.

Если нужен multi-replica shared filesystem:
- storage должен поддерживать RWX;
- application должна быть concurrency-safe.

## Failure simulations

1. Wrong StorageClass -> PVC Pending -> Pod Pending.
2. PVC не создан -> Pod Pending.
3. Scale replicas=2 с RWOP -> второй Pod не сможет использовать claim.
4. Disk full -> Pod Running, application writes fail.
5. Delete Pod -> replacement монтирует тот же PVC, data остаётся.
