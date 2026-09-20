# Walkthrough — Blue/Green Deployment

## Что строим

```text
orders-blue  = current
orders-green = candidate

Service orders
     |
selector color=blue
     |
     v
 blue Pods
```

После cutover selector меняется на `green`.

## Файлы

- [README](README.md)
- [Полный manifest](all.yaml)
- [Deployment reference](../../docs/knowledge/manifests/deployment.md)
- [Service reference](../../docs/knowledge/manifests/service.md)

## Почему два labels

```yaml
app: orders
color: blue
```

`app=orders` — stable application identity.

`color=blue|green` — slot.

Это позволяет оставить Service name постоянным.

## Service mapping

```yaml
selector:
  app: orders
  color: blue
```

Production traffic идёт только в blue.

Preview Service выбирает green:

```text
orders-green-preview
 -> app=orders,color=green
```

Так green можно smoke-test до cutover.

## Cutover

```bash
kubectl patch svc orders   -p '{"spec":{"selector":{"app":"orders","color":"green"}}}'
```

Runtime:

```text
Service selector changes
 -> EndpointSlice recomputed
 -> blue endpoints disappear
 -> green endpoints appear
 -> new traffic goes green
```

Pod restart не нужен.

## Важно: switch не атомарен для всего мира

Могут существовать:
- in-flight requests;
- keep-alive connections;
- proxy state.

Поэтому blue и green должны быть безопасны одновременно.

## DB compatibility

Плохой сценарий:

```text
green migration drops column
 -> blue still running
 -> blue SQL fails
```

Нужен expand-contract:

```text
add compatible schema
 -> deploy green
 -> validate
 -> switch
 -> remove blue later
 -> contract schema later
```

## Resource capacity

Blue/green временно удваивает workload.

Пример:

```text
blue 6 Pods × 512Mi
green 6 Pods × 512Mi
≈ 6Gi requested memory
```

Cluster должен выдерживать обе версии.

## Rollback

```bash
kubectl patch svc orders   -p '{"spec":{"selector":{"app":"orders","color":"blue"}}}'
```

Но это не откатывает DB/events/external effects.

## Где быть внимательным

### Too broad selector

Если:

```yaml
selector:
  app: orders
```

то Service выберет и blue, и green одновременно.

Это уже mixed-version routing, а не blue/green.

### Green Ready != business correct

Readiness может быть зелёной, но:
- wrong feature flag;
- wrong DB migration;
- wrong integration config
могут обнаружиться только smoke/business test.

## Проверка

```bash
kubectl get pod -L app,color
kubectl get svc orders -o yaml
kubectl get endpointslice   -l kubernetes.io/service-name=orders
```
