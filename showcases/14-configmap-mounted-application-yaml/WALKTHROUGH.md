# Walkthrough — ConfigMap mounted as application.yaml

## Архитектура

```text
ConfigMap
 application.yaml
      |
      v
Kubernetes volume
      |
      v
/etc/spring/application.yaml
      |
      v
Spring config loader
      |
      v
Spring Environment
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [ConfigMap reference](../../docs/knowledge/manifests/configmap.md)

## Почему file config

Environment variables удобны для flat config.

YAML удобнее для:
- nested structures;
- lists;
- maps;
- readable groups.

## ConfigMap key becomes file

```yaml
data:
  application.yaml: |
    app:
      customer-api:
        base-url: http://customer-api:8080
```

При volume mount:

```text
/etc/spring/application.yaml
```

## Volume mapping

```text
ConfigMap.metadata.name
      =
volumes[].configMap.name

volumes[].name
      =
volumeMounts[].name
```

Это две разные связи.

## Spring loading

```yaml
SPRING_CONFIG_ADDITIONAL_LOCATION: file:/etc/spring/
```

Spring Boot добавляет directory к configuration search.

## Immutable config versioning

В showcase:

```yaml
metadata:
  name: orders-application-config-v1
immutable: true
```

Следующая версия:

```text
orders-application-config-v2
```

Deployment reference изменяется -> rollout.

Плюсы:
- audit;
- rollback;
- no surprise mutable config.

## Update semantics

Mutable ConfigMap volume eventually обновит file.

Но Spring already loaded config into memory.

```text
ConfigMap changed
 -> mounted file may change
 -> Spring beans do NOT magically rebuild
```

Поэтому versioned ConfigMap + rollout — простой production pattern.

## subPath warning

Если использовать `subPath`:
automatic ConfigMap file updates не приходят.

## Failure scenarios

### Invalid YAML
Pod container starts, Spring exits.

### Missing ConfigMap
Container config setup fails before app startup.

### Wrong property type
Spring binding/validation fails.

### File changed but app not restarted
Filesystem and in-memory config diverge.

## Проверка

```bash
kubectl get configmap orders-application-config-v1 -o yaml
kubectl describe pod <pod>
kubectl exec <pod> -- cat /etc/spring/application.yaml
```

## Главное

Kubernetes умеет доставить file. Reload application configuration — отдельная Spring/application responsibility.
