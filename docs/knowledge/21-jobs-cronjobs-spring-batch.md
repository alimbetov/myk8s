# 21 — Jobs / CronJobs / Spring Batch

## Учебная карта темы

### Кто за что отвечает

```text
CronJob
  -> schedule

Job
  -> completion / retry / deadline

Pod
  -> process execution

Spring Batch
  -> steps / chunks / checkpoints / business restartability
```

### Почему не всегда @Scheduled

```text
Deployment replicas=4
   ↓
@Scheduled in each JVM
   ↓
potential 4 executions
```

CronJob централизует scheduling, но idempotency всё равно остаётся application responsibility.

### Annotated fragment

```yaml
schedule: "0 2 * * *"
timeZone: Asia/Almaty
concurrencyPolicy: Forbid

jobTemplate:
  spec:
    backoffLimit: 2
    activeDeadlineSeconds: 3600
```

Практика: [Showcase 04](../../showcases/04-batch-cronjob/README.md).

## Для аналитика, разработчика и тестировщика

### Аналитик

Фиксирует schedule, timezone, overlap policy, deadline, retry и business idempotency.

### Разработчик

Разделяет Kubernetes Job retry и Spring Batch restartability/checkpoints.

### Тестировщик

Проверяет:
- duplicate execution;
- overlap;
- missed schedule;
- non-zero exit;
- retry;
- timeout;
- manual rerun.

### Перед следующей главой

Нужно уметь различать:

```text
schedule
execution retry
business restartability
idempotency
```

Проверено: 2026-09-20.

Не каждое Spring Boot приложение должно работать вечно. Batch task лучше моделировать как завершённую работу.

## 1. Deployment vs Job

Deployment:

```text
desired: process должен работать постоянно
```

Job:

```text
desired: task должна успешно завершиться
```

Это разные lifecycle semantics.

## 2. Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: daily-reconciliation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: job
          image: registry/reconciliation:1.2.0
```

Controller отслеживает successful completion.

## 3. restartPolicy

Для Job обычно `Never` или `OnFailure`, в зависимости от design.

Не путать container restart с Job retry/new Pod behavior.

## 4. backoffLimit

```yaml
backoffLimit: 3
```

Ограничивает retry attempts Job после failures.

Но application side effects должны быть idempotent.

## 5. Idempotency

Если task:
1. записала половину rows;
2. упала;
3. Job retry;

нужно понимать, что произойдёт повторно.

Подходы:
- checkpoint;
- unique business key;
- transaction boundaries;
- idempotency markers;
- Spring Batch JobRepository.

## 6. Spring Batch

Spring Batch предоставляет:
- Job;
- Step;
- ItemReader/Processor/Writer;
- restartability;
- metadata repository;
- chunk transactions.

Kubernetes Job отвечает за compute lifecycle. Spring Batch — за batch processing semantics.

Они дополняют друг друга.

## 7. CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: report
              image: registry/report:1.0.0
```

CronJob создаёт Jobs по schedule.

## 8. Timezone

CronJob schedule/timezone следует задавать осознанно. Не предполагаете автоматически timezone node/application.

Для бизнеса в Казахстане важно явно определить business timezone в requirements и CronJob configuration/application logic.

## 9. concurrencyPolicy

Важные значения:
- Allow;
- Forbid;
- Replace.

Если hourly task работает 90 минут:

```text
12:00 Job still running
13:00 next schedule
```

Что делать — business decision.

`Forbid` не запускает concurrent Job.

## 10. startingDeadlineSeconds

Определяет, насколько поздно допустимо начать missed scheduled execution.

Полезно после control-plane downtime.

## 11. successfulJobsHistoryLimit / failedJobsHistoryLimit

Ограничивают history objects, чтобы namespace не захламлялся Jobs.

Logs/history нужно хранить через observability, а не бесконечными Job objects.

## 12. TTL after finished

Job cleanup можно автоматизировать TTL controller:

```yaml
ttlSecondsAfterFinished: 3600
```

## 13. ConfigMap/Secret

Batch job получает config так же, как Deployment:
- env;
- ConfigMap;
- Secret.

Но credential validity должен покрывать duration job.

## 14. Resources

Batch может быть CPU/memory intensive. Requests нужны, иначе большой Job может destabilize node.

Можно использовать отдельные nodes/priority/quotas по platform policy.

## 15. Spring scheduling inside Deployment

```java
@Scheduled(cron = "...")
```

При replicas=5 scheduler сработает в пяти JVM, если нет distributed locking.

Это частая ошибка migration монолита в Kubernetes.

## 16. Когда @Scheduled допустим

Если:
- задача может выполняться в каждой replica;
- или используется reliable distributed lock;
- или leader election.

Для true singleton periodic batch CronJob часто проще.

## 17. Failure practice: duplicate execution

Deployment replicas=2 + `@Scheduled`.

Наблюдайте два запуска.

Это architecture bug, Kubernetes работает правильно.

## 18. Failure practice: Job retry duplicates side effect

Job пишет external request, затем падает до checkpoint.

Retry повторяет request.

Решение: idempotency key/outbox/state tracking.

## 19. Failure practice: Cron overlap

Долгий Job + `concurrencyPolicy: Allow`.

Observe parallel Jobs.

Change to Forbid if business semantics require.

## 20. Graceful termination

Batch Job может быть evicted/node drain.

Spring Batch restartability/checkpoint design важнее надежды «Job никогда не прервут».

## 21. Observability

Нужно знать:
- started/completed/failed;
- duration;
- records read/written/skipped;
- retry count;
- last successful business date.

`Job Complete` не обязательно означает business data correct — добавляйте domain metrics.

## 22. Anti-patterns

- singleton `@Scheduled` in multi-replica Deployment without lock;
- infinite Job retry;
- non-idempotent retry;
- no resource requests;
- no cleanup;
- store batch progress only in Pod filesystem.

## 23. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| batch semantics | ✓ | | |
| Job/CronJob manifest | ✓ | | ✓ |
| schedule | ✓ | | ✓ |
| resource policy | | ✓ | ✓ |
| idempotency | ✓ | | |
| monitoring | | | ✓ |

## 24. CKAD

Jobs/CronJobs — важный CKAD primitive. Spring Batch semantics — production/application engineering.

## Связанные production-like примеры

- [CronJob -> Job -> Spring Batch](../../showcases/04-batch-cronjob/README.md)

Для каждого стенда откройте `README.md` → `WALKTHROUGH.md` → `all.yaml`.

## Sources

- https://kubernetes.io/docs/concepts/workloads/controllers/job/
- https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
- https://docs.spring.io/spring-batch/reference/

Проверено: **2026-09-20**.
