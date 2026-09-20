# 06 — CKAD map vs production engineering

Проверено: 2026-09-20.

На дату проверки CKAD exam page указывает Kubernetes **v1.35** и 2-часовой performance-based exam.

## Domains

| Domain | Weight |
|---|---:|
| Application Design and Build | 20% |
| Application Deployment | 20% |
| Application Observability and Maintenance | 15% |
| Application Environment, Configuration and Security | 25% |
| Services and Networking | 20% |

## Нужно делать быстро

```bash
kubectl create deployment
kubectl expose deployment
kubectl set image
kubectl rollout status
kubectl rollout undo
kubectl create configmap
kubectl create secret generic
kubectl logs
kubectl logs --previous
kubectl exec
kubectl describe
kubectl get events
kubectl run
kubectl create job
kubectl create cronjob
kubectl auth can-i
```

Уметь без долгого поиска:
- Deployment;
- Pod;
- Job/CronJob;
- Service;
- ConfigMap/Secret consumption;
- probes;
- requests/limits;
- volume/PVC;
- SecurityContext;
- ServiceAccount;
- NetworkPolicy;
- Ingress;
- init/sidecar patterns.

## CKAD != production completeness

За пределами экзамена или значительно глубже:
- external secret management and rotation;
- HA database/message broker operations;
- backup/restore/PITR;
- PKI lifecycle;
- service mesh/workload identity;
- SLO/alerting;
- capacity planning;
- DR/RPO/RTO;
- operator lifecycle;
- safe DB/message schema evolution.

## Training method

Каждая production lab должна иметь CKAD-shortcut:

```text
production version -> understand why
minimal CKAD version -> type fast
break it -> diagnose
repair it -> verify
```

## Source

- https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/

Проверено: **2026-09-20**.
