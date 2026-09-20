# Showcase 09 — Canary with Gateway API weighted backends

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
Gateway
  |
HTTPRoute
  |
  +-- weight 90 --> Service orders-stable --> v1 Pods
  |
  +-- weight 10 --> Service orders-canary --> v2 Pods
```

## Почему два Service

Каждый Service выбирает только одну version:

```text
orders-stable selector version=v1
orders-canary selector version=v2
```

HTTPRoute занимается traffic weight.

## Mapping

```yaml
backendRefs:
  - name: orders-stable
    port: 8080
    weight: 90
  - name: orders-canary
    port: 8080
    weight: 10
```

Weights являются relative proportions, а не обещанием идеального распределения каждого короткого окна.

## Promotion

Постепенно:

```text
90/10
75/25
50/50
0/100
```

только после оценки:
- 5xx;
- latency;
- business failures;
- JVM/DB metrics.

## Rollback

Установить canary weight 0 / stable 100 либо удалить canary backend according to rollout tool/process.

## State compatibility

Stable и canary работают одновременно, поэтому:
- DB schema backward-compatible;
- messages compatible;
- cache format compatible;
- external side effects idempotent.

## Prerequisite

Gateway API implementation должна поддерживать weighted HTTPRoute backends.

## Failure simulations

1. Canary Pods NotReady -> backend degraded.
2. Weight 100 canary без проверки -> фактически full rollout.
3. v2 breaking DB migration -> v1 тоже ломается.
4. Metrics aggregate без version label -> canary regression трудно увидеть.
