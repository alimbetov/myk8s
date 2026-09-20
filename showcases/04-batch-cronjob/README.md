# Showcase 04 — Spring Batch as Kubernetes CronJob

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Место этого стенда в общей системе

```text
Kubernetes scheduler/controller
        |
        v
CRONJOB   ← расписание
        |
        v
JOB       ← completion/retry
        |
        v
POD
        |
        v
Spring Batch
        |
        v
DB / files / external APIs
```

CronJob используется вместо постоянно работающего Deployment, когда задача должна запускаться **по расписанию и завершаться**.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
Cron schedule
    |
    v
CronJob
    |
    v
Job
    |
    v
Pod
    |
    +--> ConfigMap
    +--> Secret
    +--> PostgreSQL / external systems
    |
    v
exit 0 / non-zero
```

## Почему не Deployment + @Scheduled

Если Deployment имеет 4 replicas:

```text
Pod1 @Scheduled
Pod2 @Scheduled
Pod3 @Scheduled
Pod4 @Scheduled
```

без distributed lock задача может выполниться 4 раза.

CronJob делает scheduling обязанностью Kubernetes.

## Mapping

### CronJob.jobTemplate -> Job

Всё внутри:

```yaml
jobTemplate:
  spec:
    backoffLimit: 2
    template:
      spec:
        ...
```

становится spec создаваемого Job.

### Job Pod -> Spring profile

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: batch
```

### concurrencyPolicy

```yaml
concurrencyPolicy: Forbid
```

если предыдущий Job ещё работает, concurrent execution не стартует.

Это не заменяет application idempotency: duplicate execution всё равно должен быть безопасен при retries/manual runs.

## Business timezone

```yaml
timeZone: Asia/Almaty
```

делает business scheduling явным.

## Failure simulations

1. Exit code 1 -> Job retries до backoffLimit.
2. Task работает дольше schedule -> сравнить Allow и Forbid.
3. После side effect crash -> показать необходимость idempotency.
4. Missing Secret -> Pod не стартует.
5. Job завис -> activeDeadlineSeconds завершает execution.
