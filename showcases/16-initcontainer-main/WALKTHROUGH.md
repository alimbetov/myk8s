# Walkthrough — initContainer + main container

## Архитектура

```text
ConfigMap template
      |
      v
initContainer prepare-config
      |
      v
shared emptyDir
      |
      v
generated application.yaml
      |
      v
Spring Boot main container
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [Pod reference](../../docs/knowledge/manifests/pod.md)

## Init ordering

Kubernetes запускает ordinary init containers последовательно.

Main containers не стартуют, пока init container не завершится успешно.

## Downward API

Showcase получает Pod name:

```yaml
valueFrom:
  fieldRef:
    fieldPath: metadata.name
```

Init container подставляет это значение в generated config.

## Shared emptyDir

```text
init mount /work/config
       |
       | same volume name
       v
main mount /work/config
```

`emptyDir` живёт в lifecycle Pod.

Pod replacement -> новый emptyDir -> init выполняется заново.

## Runtime sequence

```text
Pod scheduled
 -> mount ConfigMap + emptyDir
 -> start prepare-config
 -> generate file
 -> init exits 0
 -> start Spring app
 -> Spring loads generated file
```

## Где полезно

- generate config;
- unpack artifacts;
- filesystem preparation;
- one-time local bootstrap.

## Где опасно

### DB migration
5 replicas могут одновременно запускать migration init container.

### Infinite dependency wait
Pod будет в init forever, скрывая реальный dependency problem.

Лучше finite retry + readiness where appropriate.

## Failure scenarios

### Init exit 1
Main never starts.

### ConfigMap missing
Init cannot start/mount.

### Generated YAML invalid
Init succeeds, Spring fails.

### Wrong shared volume name
Main не видит generated file.

## Диагностика

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl logs <pod> -c prepare-config
kubectl logs <pod> -c app
```

## Главное

InitContainer — preparation phase Pod lifecycle, не универсальный orchestration engine.
