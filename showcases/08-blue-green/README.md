# Showcase 08 — Blue/Green Deployment

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Полный связный manifest](all.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
                 +--> Deployment orders-blue
Service orders --|
                 +--> Deployment orders-green
```

Но Service одновременно выбирает **только один color** через selector.

## Labels

Оба Deployment имеют stable application label:

```text
app=orders
```

и version slot:

```text
color=blue
color=green
```

Service:

```yaml
selector:
  app: orders
  color: blue
```

Cutover:

```yaml
selector:
  app: orders
  color: green
```

## Почему это работает

Service selector динамически меняет selected Pods; EndpointSlice controller обновляет backends.

```text
Service selector color=blue
 -> blue endpoints

patch Service

Service selector color=green
 -> green endpoints
```

## Проверка green до cutover

Создан отдельный preview Service:

```text
orders-green-preview
```

Он всегда выбирает green Pods.

Так smoke tests можно выполнить без production traffic switch.

## Команда cutover

```bash
kubectl patch svc orders   -p '{"spec":{"selector":{"app":"orders","color":"green"}}}'
```

Rollback:

```bash
kubectl patch svc orders   -p '{"spec":{"selector":{"app":"orders","color":"blue"}}}'
```

## Что Service switch НЕ rollback'ит

- DB migration;
- emitted Kafka events;
- external API calls;
- cache mutations;
- secret rotation.

Поэтому blue/green требует backward-compatible state.

## Failure simulations

1. green readiness fails -> preview endpoints unavailable.
2. Service selector typo -> no endpoints.
3. green uses incompatible DB schema -> traffic switch exposes application error.
4. both colors accidentally selected by too-broad selector -> versions mix.

## Production note

Для сложного traffic switching можно использовать Gateway API/progressive delivery controller. Этот стенд показывает самый простой Kubernetes-native selector switch.
