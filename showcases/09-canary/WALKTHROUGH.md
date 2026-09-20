# Walkthrough — Canary Deployment

## Архитектура

```text
               HTTPRoute
                /     \
             90%       10%
              |          |
 orders-stable      orders-canary
      |                   |
  version=v1          version=v2
      |                   |
   v1 Pods             v2 Pods
```

## Файлы

- [README](README.md)
- [Полный manifest](all.yaml)
- [Deployment strategies](../../docs/knowledge/19-deployment-strategies.md)

## Почему два Service

Service сам по себе не является percentage traffic controller.

```text
orders-stable -> only v1
orders-canary -> only v2
```

Weighted decision делает HTTPRoute.

## Weight

```yaml
backendRefs:
  - name: orders-stable
    weight: 90
  - name: orders-canary
    weight: 10
```

Это relative proportion.

Не ожидайте идеального 9/1 распределения на каждых десяти requests.

## Replica count != weight

```text
stable replicas=10
canary replicas=1
```

не означает 90/10.

Replica count = capacity.
Weight = routing proportion.

## Canary capacity

Если на canary приходит 10% от 5000 rps:

```text
500 rps canary
/ 1 Pod
= 500 rps per Pod
```

Возможно, canary нужен не один Pod.

## Observability

Canary бессмыслен, если metrics нельзя разделить по version.

Нужны labels/tags:

```text
version=v1
version=v2
```

Сравнивать:
- 5xx;
- p95/p99;
- business failures;
- DB errors;
- JVM saturation.

## Promotion

Пример последовательности:

```text
99/1
95/5
90/10
75/25
50/50
0/100
```

Это пример, не обязательная схема.

## Совместимость

v1 и v2 работают одновременно.

Нужны compatible:
- DB schema;
- Kafka events;
- cache format;
- API contracts.

## Failure scenarios

### v2 5xx
Уменьшить weight/rollback.

### v2 NotReady
Canary Service не имеет ready endpoints.

### Breaking migration
Rollback routing не вернёт старую DB schema.

## Проверка

```bash
kubectl get httproute orders-canary -o yaml
kubectl get svc orders-stable orders-canary
kubectl get endpointslice
kubectl get pod -L version
```

## Production

Часто progressive delivery controller автоматизирует:
- weight changes;
- metric analysis;
- rollback.

Этот стенд специально показывает underlying Kubernetes/Gateway mechanics.
