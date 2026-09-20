# Manifest Reference — Job + CronJob

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Job purpose
Run work to successful completion.

## Job apiVersion / kind
- apiVersion: batch/v1
- kind: Job

## Job spec core
- template
- backoffLimit
- completions
- parallelism
- completionMode
- activeDeadlineSeconds
- ttlSecondsAfterFinished
- suspend

## Pod template
- restartPolicy
- containers
- resources
- env/config
- serviceAccountName

## CronJob purpose
Create Jobs on schedule.

## CronJob core
- schedule
- timeZone
- concurrencyPolicy
- startingDeadlineSeconds
- successfulJobsHistoryLimit
- failedJobsHistoryLimit
- suspend
- jobTemplate

## Runtime concepts
- retries vs Pod restart
- idempotency
- overlap
- missed schedules
- cleanup

## Spring Batch link
Kubernetes Job = compute lifecycle; Spring Batch = batch processing semantics/checkpoints.

## Troubleshooting
```bash
kubectl get cronjob,job,pod
kubectl describe job <name>
kubectl logs <pod>
kubectl create job --from=cronjob/<name> manual-run
```

## Sources
- https://kubernetes.io/docs/concepts/workloads/controllers/job/
- https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
