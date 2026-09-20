# Walkthrough — Spring Batch as CronJob

## Архитектура

```text
Cron schedule
   ↓
CronJob
   ↓
Job
   ↓
Pod
   ↓
Spring Batch
   ↓
exit code
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [Job/CronJob reference](../../docs/knowledge/manifests/job-cronjob.md)

## CronJob vs @Scheduled

Deployment replicas multiply `@Scheduled` executions unless locking/leader election exists.

CronJob centralizes scheduling in Kubernetes.

## Mapping

```text
CronJob.spec.jobTemplate
   ↓
Job.spec
   ↓
Job.spec.template
   ↓
Pod
```

Schedule/concurrency are CronJob concerns.

Retry/deadline are Job concerns.

## timeZone

```yaml
timeZone: Asia/Almaty
```

Business schedule становится explicit.

## concurrencyPolicy

`Forbid` не запускает новый scheduled Job, если previous execution ещё active.

Но manual duplicate или retry всё равно возможны — application idempotency необходима.

## activeDeadlineSeconds

Ограничивает общий runtime Job.

Это защита от зависших batch executions.

## Failure scenarios

- exit code !=0 -> retries;
- previous run still active -> Forbid skip;
- side effect happened before crash -> duplicate on retry;
- Secret missing -> Pod cannot start;
- task hangs -> active deadline.

## Проверка

```bash
kubectl get cronjob,job,pod
kubectl describe cronjob daily-reconciliation
kubectl describe job <job>
kubectl logs <pod>
```

## Production

Храните business observability:
- records processed;
- business date;
- skipped/errors;
- last successful run.

Job Complete != business data correct.
