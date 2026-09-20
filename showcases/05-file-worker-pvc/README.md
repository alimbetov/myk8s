# Showcase 05 — File-processing worker with PVC

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Место этого стенда в общей системе

```text
Deployment / Pod
      |
      v
volumeMount
      |
      v
PVC       ← application requests storage here
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

PVC находится **ниже Pod**, а не внутри Spring Boot. Приложение видит обычный filesystem path, а Kubernetes связывает его с storage backend.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
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
