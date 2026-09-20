# Showcase 04 — Spring Batch as Kubernetes CronJob

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Полный связный manifest](all.yaml)
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
