# Walkthrough — StatefulSet + Headless Service

## Архитектура

```text
Headless Service cluster-member
         |
         +--> cluster-member-0.cluster-member
         +--> cluster-member-1.cluster-member
         +--> cluster-member-2.cluster-member

StatefulSet
   |
   +--> ordinal 0 -> PVC data-cluster-member-0
   +--> ordinal 1 -> PVC data-cluster-member-1
   +--> ordinal 2 -> PVC data-cluster-member-2
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [PVC reference](../../docs/knowledge/manifests/pvc.md)

## Headless Service

```yaml
clusterIP: None
```

Это отключает обычный virtual ClusterIP load-balancing model.

DNS может вернуть individual Pod addresses.

## serviceName

StatefulSet:

```yaml
serviceName: cluster-member
```

должен соответствовать governing headless Service name.

## Stable identity

Deployment Pod names случайны.

StatefulSet:

```text
cluster-member-0
cluster-member-1
cluster-member-2
```

Ordinal сохраняет logical identity при replacement.

## volumeClaimTemplates

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
```

Container:

```yaml
volumeMounts:
  - name: data
```

Mapping по name.

Каждый ordinal получает собственный PVC.

## Replacement

Удалить Pod 1:

```text
cluster-member-1 deleted
 -> controller creates cluster-member-1
 -> same logical identity
 -> same PVC associated with ordinal
```

## OrderedReady

Default Pod management:

```text
0 Ready
 -> start 1
1 Ready
 -> start 2
```

Это может быть полезно stateful systems, но может замедлять recovery/startup.

## StatefulSet не делает HA

Он не знает semantics вашего distributed database.

```text
StatefulSet:
 identity + ordering + storage

Product/operator:
 replication + leader election + quorum + failover
```

Именно поэтому Kafka/PostgreSQL/RabbitMQ отдельные operator showcases.

## Failure scenarios

### Pod 0 NotReady
Ordered rollout/start may stall later ordinals.

### PVC Pending
Pod cannot become usable.

### Wrong serviceName
Stable peer DNS architecture broken.

### Scale down
Pod disappears, PVC normally remains unless retention configured otherwise.

## Диагностика

```bash
kubectl get sts
kubectl describe sts cluster-member
kubectl get pod
kubectl get svc cluster-member
kubectl get endpointslice
kubectl get pvc
```

## Главное

StatefulSet — Kubernetes identity/storage primitive, не готовая distributed database platform.
