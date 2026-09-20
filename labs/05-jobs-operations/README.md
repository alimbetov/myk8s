# Lab 05 — Jobs, CronJobs and operational diagnostics

## Цель

На практике различить Deployment и Job lifecycle, увидеть CronJob overlap и построить маленький operational runbook.

## 1. CronJob

Скопируйте пример и замените image:

```bash
kubectl apply -f ../../examples/spring-boot/batch/cronjob.yaml
kubectl get cronjob
```

Для быстрой проверки можно временно поставить schedule каждую минуту.

## 2. Наблюдение

```bash
kubectl get cronjob,job,pod -w
```

Проследите цепочку:

```text
CronJob
 -> Job
 -> Pod
 -> process exits
 -> Job Complete
```

## 3. Manual Job from CronJob

```bash
kubectl create job --from=cronjob/spring-batch-demo manual-run
```

Проверить logs.

## 4. Failure — non-zero exit

Используйте image/command, который завершится exit code !=0.

Наблюдайте:
- retries;
- `backoffLimit`;
- Job Failed.

## 5. Failure — overlap

Сделайте task дольше schedule interval и сравните:
- `concurrencyPolicy: Allow`;
- `Forbid`.

Объясните business effect duplicates.

## 6. Operational runbook

Для failed Job:

```bash
kubectl get job
kubectl describe job <job>
kubectl get pod -l job-name=<job>
kubectl logs <pod>
kubectl get events --sort-by=.lastTimestamp
```

Root cause должен быть конкретным:
- scheduling;
- image;
- configuration;
- application exit;
- dependency.

## 7. Observability questions

Для batch недостаточно знать container exit code.

Добавьте/определите metrics:
- records processed;
- failed;
- duration;
- business date;
- last successful run.

## Cleanup

Удалите lab namespace.
