# Manifest Reference — Job + CronJob

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

Этот файл покрывает два связанных kinds:
- Job
- CronJob

# Part A — Job

## 1. Назначение

`Job` запускает Pods до успешного завершения требуемой работы.

Это completion-oriented workload, в отличие от Deployment, где process должен жить постоянно.

## 2. API

```yaml
apiVersion: batch/v1
kind: Job
```

## 3. Minimal Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: reconciliation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: job
          image: registry.example/reconciliation:1.0.0
```

## 4. backoffLimit

```yaml
backoffLimit: 3
```

Количество retries перед переводом Job в failed.

**Default: 6**, если не используется per-index backoff configuration для Indexed Job.

Retries создаются с exponential backoff, который увеличивается между attempts.

## 5. completions

Сколько successful Pod completions нужно для завершения Job.

Если не указан вместе с parallelism defaults зависят от Job mode; для обычного non-parallel Job ожидается одна successful completion.

## 6. parallelism

Максимальное число Pods, работающих параллельно для Job.

```yaml
parallelism: 4
```

Не увеличивать без оценки:
- DB writes;
- external API rate limits;
- partitioning/idempotency.

## 7. completionMode

Основные варианты:
- NonIndexed
- Indexed

Indexed mode назначает Pods completion indexes и полезен для partitioned work.

## 8. activeDeadlineSeconds

Maximum runtime Job overall.

Когда deadline превышен, Job считается failed и active Pods завершаются.

Deadline Job имеет precedence над retry loop: infinite retries не могут обойти общий active deadline.

## 9. ttlSecondsAfterFinished

```yaml
ttlSecondsAfterFinished: 3600
```

TTL controller может удалить finished Job автоматически после указанного количества секунд.

Полезно для cleanup, но logs/metrics должны уже быть централизованы.

## 10. suspend

```yaml
suspend: true
```

Приостанавливает Job execution scheduling до resume.

## 11. template.spec.restartPolicy

Для Job допустимы:
- Never
- OnFailure

`Always` не используется как обычная Job policy.

Выбор влияет на то, retry происходит как restart container в том же Pod или replacement Pods/controller accounting.

## 12. podFailurePolicy

Advanced field для granular reaction на Pod failures:
- FailJob
- Ignore
- Count
- FailIndex и др. в актуальных APIs.

Использовать, когда exit codes / disruption reasons требуют разных actions.

## 13. successPolicy

Современный Kubernetes поддерживает success policy для indexed Jobs, позволяя считать Job успешным до завершения всех indexes при заданных rules.

Advanced batch orchestration feature.

## 14. Idempotency

Kubernetes может повторить task.

Application обязан учитывать:

```text
side effect happened
 -> process crashed before controller observed success
 -> retry may happen
```

Нужны:
- idempotency keys;
- checkpoint;
- unique constraints;
- Spring Batch metadata;
- transactional boundaries.

## 15. Spring Batch relation

```text
Kubernetes Job
 -> compute scheduling/lifecycle

Spring Batch
 -> steps/chunks/checkpoints/restartability/business processing
```

Они дополняют друг друга.

## 16. Failure — non-zero exit

```bash
kubectl get job
kubectl describe job reconciliation
kubectl get pod -l job-name=reconciliation
kubectl logs <pod>
```

Следить за retry count и backoffLimit.

## 17. Failure — duplicate side effect

External call выполнен, process упал до durable success marker.

Retry повторяет call.

Это application idempotency failure.

## 18. Failure — active deadline

Job всё ещё Running, но total execution > activeDeadlineSeconds.

Controller завершает workload и Job fails.

# Part B — CronJob

## 19. API

```yaml
apiVersion: batch/v1
kind: CronJob
```

## 20. Minimal CronJob

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
              image: registry.example/report:1.0.0
```

## 21. schedule

Cron syntax.

Controller interprets schedule according to `timeZone` if specified, otherwise controller local timezone.

Production business jobs should set timezone explicitly where business date matters.

## 22. timeZone

Stable since Kubernetes 1.27.

```yaml
timeZone: "Asia/Almaty"
```

Использовать valid IANA timezone name.

Не добавлять CRON_TZ/TZ в schedule expression вместо field.

## 23. concurrencyPolicy

Values:
- Allow
- Forbid
- Replace

**Default: Allow.**

### Allow
Concurrent Jobs from same CronJob allowed.

### Forbid
If previous run still active, new scheduled run skipped.

### Replace
Current active Job replaced by new run.

Policy относится только к Jobs этого конкретного CronJob.

## 24. startingDeadlineSeconds

Если scheduled run был пропущен, задаёт максимальную lateness, при которой controller ещё может его создать.

Если field отсутствует — explicit deadline нет.

Слишком маленькое значение может приводить к skipped executions при controller delays.

## 25. suspend

```yaml
suspend: true
```

**Default: false.**

Новые scheduled executions не запускаются. Уже started Jobs не останавливаются.

Unsuspend может привести к запуску missed Jobs, особенно если starting deadline отсутствует.

## 26. successfulJobsHistoryLimit

**Default: 3.**

`0` означает не хранить finished successful Jobs.

## 27. failedJobsHistoryLimit

**Default: 1.**

`0` — не хранить failed history objects.

## 28. jobTemplate

Полный Job spec template.

То есть CronJob policies и Job retry/deadline semantics независимы и вложены друг в друга.

## 29. CronJob timing precision

CronJob не следует воспринимать как hard real-time scheduler.

Controller reconciliation/time delays возможны.

Business tasks должны быть tolerant to delayed/duplicate scheduling согласно documented semantics.

## 30. Failure — overlap

Task выполняется 90 мин, schedule каждый час, concurrencyPolicy=Allow.

Получаем simultaneous executions.

Если business forbids overlap -> Forbid/locking/idempotency.

## 31. Failure — missed schedule

Controller downtime/delay больше startingDeadlineSeconds -> run skipped.

Проверять CronJob events/status.

## 32. Failure — timezone misunderstanding

Business expects Almaty 02:00, controller interprets default timezone otherwise.

Set `timeZone: Asia/Almaty` explicitly.

## 33. Day-2 commands

```bash
kubectl get job
kubectl get cronjob
kubectl describe job <name>
kubectl describe cronjob <name>
kubectl get pods -l job-name=<job>
kubectl logs <pod>
kubectl create job --from=cronjob/nightly-report manual-run
kubectl patch cronjob nightly-report -p '{"spec":{"suspend":true}}'
```

## 34. Anti-patterns

- @Scheduled singleton assumption in multi-replica Deployment;
- no idempotency;
- huge backoffLimit;
- no active deadline for unbounded tasks;
- Allow when overlap destructive;
- no explicit timezone for business calendar jobs;
- rely on Job objects for long-term history.

## 35. CKAD

Must know:
- create Job/CronJob;
- restartPolicy;
- schedule;
- backoffLimit;
- completions/parallelism;
- suspend;
- logs/describe;
- manual Job from CronJob.

## Sources

- https://kubernetes.io/docs/concepts/workloads/controllers/job/
- https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/job-v1/
- https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
- https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/cron-job-v1/
