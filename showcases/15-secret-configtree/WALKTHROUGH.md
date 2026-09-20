# Walkthrough — Secret as Spring Boot configtree

## Архитектура

```text
Kubernetes Secret
  spring.datasource.username
  spring.datasource.password
          |
          v
mounted files
          |
          v
/etc/secrets/
          |
          v
configtree import
          |
          v
Spring Environment
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [Secret reference](../../docs/knowledge/manifests/secret.md)

## Почему configtree

Secret values не обязаны попадать в process environment.

Каждый key становится file.

Например:

```text
/etc/secrets/spring.datasource.username
/etc/secrets/spring.datasource.password
```

## Spring import

```text
SPRING_CONFIG_IMPORT=configtree:/etc/secrets/
```

Spring использует filename как property key.

## Mapping

```text
Secret.metadata.name
      =
volume.secret.secretName

volume.name
      =
volumeMount.name

mountPath
      =
configtree directory
```

## Что остаётся env

DB URL не secret:

```text
SPRING_DATASOURCE_URL
```

Поэтому он приходит из ConfigMap.

Это хороший split:

```text
routing/config -> ConfigMap
credentials    -> Secret/configtree
```

## Rotation

Secret volume eventually updates files.

Но Hikari pool может продолжать:
- existing authenticated connections;
- old in-memory password.

Rotation runbook:

```text
new credential valid
 -> Secret updated
 -> app reload/rollout
 -> verify new connections
 -> revoke old
```

## Failure scenarios

### Missing Secret
Pod cannot mount required volume.

### Missing filename/property
Spring property unresolved.

### Wrong mountPath
Configtree sees empty/wrong directory.

### Secret changed without reload
Files and application credentials diverge.

## Security

Configtree reduces env exposure, but Secret is still accessible:
- through Kubernetes API permissions;
- inside Pod filesystem;
- through backups.

Least privilege still required.

## Проверка

```bash
kubectl describe pod <pod>
kubectl exec <pod> -- ls -la /etc/secrets
```

Не выводите password contents в shared terminal/log.

## Главное

Configtree — delivery mechanism, не complete secret lifecycle solution.
