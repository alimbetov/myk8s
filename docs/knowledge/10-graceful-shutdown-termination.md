# 10 — Graceful shutdown and termination lifecycle

## Учебная карта темы

### Что происходит при удалении Pod

```text
Pod termination requested
   ↓
Endpoint state changes / traffic draining begins
   ↓
preStop hook if configured
   ↓
SIGTERM to container process
   ↓
Spring graceful shutdown
   ↓
in-flight work completes
   ↓
process exits
   |
   └── if grace expires -> SIGKILL
```

### Временная модель

```text
terminationGracePeriodSeconds = 30s

Spring shutdown timeout
        <
Kubernetes hard grace deadline
```

### Annotated fragment

```yaml
spec:
  terminationGracePeriodSeconds: 30

# Spring:
# spring.lifecycle.timeout-per-shutdown-phase=20s
```

Для consumer это ещё важнее:

```text
stop new messages
   ↓
finish/nack current delivery
   ↓
commit/ack
   ↓
close connection
```

Практика: [RabbitMQ worker](../../showcases/11-rabbitmq-worker/README.md).

## Для аналитика, разработчика и тестировщика

### Аналитик

Для long-running requests, consumers и batch фиксирует допустимое время завершения и последствия принудительного interruption.

### Разработчик

Координирует:
- Kubernetes grace period;
- Spring shutdown timeout;
- in-flight HTTP;
- broker acknowledgements;
- DB transaction boundaries.

### Тестировщик

Проверяет termination именно во время работы:
- HTTP request;
- Rabbit/Kafka delivery;
- transaction;
- scheduled task;
- rollout/drain.

### Перед следующей главой

Нужно понимать:

```text
traffic drain
 -> SIGTERM
 -> graceful application work
 -> clean exit
 -> SIGKILL only as last resort
```

Проверено: 2026-09-20.

Zero-downtime rollout требует не только readiness нового Pod, но и корректного выключения старого.

## 1. Проблема

Плохой termination:

```text
request in progress
 -> Pod killed
 -> TCP reset
 -> user receives 5xx/error
```

Хороший termination:
- перестать принимать новую работу;
- завершить in-flight;
- commit/ack безопасно;
- закрыть resources;
- выйти до hard deadline.

## 2. Kubernetes termination

Упрощённо:

```text
Pod deletion requested
 -> endpoint termination/routing transition
 -> preStop if configured
 -> SIGTERM to container
 -> grace period counts down
 -> process exits
 -> otherwise SIGKILL
```

Не полагайтесь на мгновенное идеальное прекращение traffic: distributed propagation имеет timing.

## 3. Spring Boot graceful shutdown

Spring Boot поддерживает graceful shutdown embedded web server.

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

Kubernetes:

```yaml
terminationGracePeriodSeconds: 30
```

Оставляем запас между application timeout и platform hard deadline.

## 4. HTTP

При graceful shutdown application должен перестать принимать новую работу и дать active requests завершиться в рамках timeout.

Проверяйте реально с long-running request, а не только по документации.

## 5. preStop

`preStop` полезен только когда есть конкретная причина.

Пример:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

Но blind sleep — workaround, не архитектура. Он тратит termination grace period и требует shell.

Лучше понимать readiness/endpoint propagation и application shutdown.

## 6. Kafka consumer

Shutdown consumer — не то же самое, что HTTP.

Нужно решить:
- что с message in processing;
- когда commit offset;
- будет ли duplicate;
- как rebalance влияет на processing.

Consumer handler должен быть idempotent там, где delivery semantics допускает повтор.

## 7. RabbitMQ listener

При termination:
- stop consuming new deliveries;
- finish/ack current message;
- requeue/nack безопасно при failure;
- закрыть channel/connection.

## 8. Scheduled tasks

Если scheduler живёт в каждом replica, при 5 Pods job может выполняться 5 раз.

Для cluster-wide singleton jobs лучше:
- CronJob;
- distributed lock;
- platform scheduler;
в зависимости от semantics.

Termination scheduled task тоже должен быть продуман.

## 9. DB transactions

SIGTERM во время transaction:
- normal graceful completion может commit;
- forced kill приводит к connection close и DB rollback незавершённой transaction.

Но внешние side effects уже могли произойти. Поэтому Saga/Outbox/idempotency важнее простого graceful timeout.

## 10. terminationGracePeriodSeconds

Слишком мало:
- requests обрываются;
- consumers не успевают;
- shutdown hooks не завершаются.

Слишком много:
- bad Pod долго занимает resources;
- rollout/drain может быть медленнее.

Значение должно исходить из максимального ожидаемого безопасного work unit.

## 11. Long request anti-pattern

Если API синхронно выполняет 10-минутную операцию, termination становится сложным.

Лучше рассмотреть asynchronous job model:
- request creates task;
- worker processes;
- state stored externally.

## 12. Practical test

Запустите endpoint, который работает 10 секунд.

Параллельно:

```bash
curl http://service/slow &
kubectl delete pod <pod>
```

Наблюдайте:
- завершился request?
- появился ли replacement;
- сколько длился termination;
- были ли 5xx.

## 13. Failure practice: grace too short

Поставьте:

```yaml
terminationGracePeriodSeconds: 2
```

и 10-second request.

Ожидайте forced termination раньше завершения.

## 14. Failure practice: stuck shutdown hook

Симулируйте shutdown hook, который блокируется.

Диагностика:
- Pod долго Terminating;
- process не выходит;
- после grace будет killed.

## 15. RollingUpdate connection

```text
new Pod becomes Ready
old Pod begins termination
old Pod drains
old Pod exits
```

Если readiness/graceful shutdown неправильны, даже `maxUnavailable: 0` не гарантирует отсутствие user-visible errors.

## 16. Node drain

Drain использует eviction/termination lifecycle. Поэтому shutdown behavior тестируется и для rollout, и для maintenance.

## 17. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| HTTP graceful behavior | ✓ | | |
| consumer semantics | ✓ | | |
| grace period | | | ✓ |
| drain | | ✓ | |
| PDB | | | ✓ |
| idempotency | ✓ | | |

## 18. CKAD

Нужно понимать termination grace, lifecycle hooks, Deployment rollout. Production добавляет protocol-specific draining и idempotency.

## 19. Checklist

- SIGTERM реально доходит JVM;
- HTTP test выполнен;
- Kafka/Rabbit behavior проверен;
- grace > expected application shutdown;
- no blind huge sleeps;
- long operations externalized where appropriate;
- rollout/drain tested.

## Связанные production-like примеры

- [Batch termination semantics](../../showcases/04-batch-cronjob/README.md)
- [RabbitMQ worker graceful consumer shutdown](../../showcases/11-rabbitmq-worker/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html

Проверено: **2026-09-20**.
