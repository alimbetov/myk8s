# Kubernetes Architecture Showcases

Проверено: 2026-09-20.

`showcases/` — это ознакомительные production-like стенды. Их задача не показать отдельный объект Kubernetes, а показать **как объекты переплетаются в реальном сервисе**.

## Чем showcase отличается от examples и labs

- `examples/` — маленькие reusable manifests.
- `labs/` — упражнения, где нужно что-то сделать или сломать.
- `showcases/` — законченные архитектурные композиции, которые удобно читать как эталонную схему.

Каждый showcase отвечает на вопросы:

```text
Какие objects нужны?
Как они связаны?
Как labels попадают в selectors?
Как Service выбирает Pods?
Как Service.port связан с targetPort?
Как ConfigMap/Secret попадают в Spring Environment?
Как readiness влияет на Service?
Как NetworkPolicy меняет доступность?
Как HPA связан с requests?
Что увидит пользователь при failure?
```

## Стенды

| # | Сценарий | Главные связи |
|---|---|---|
| 01 | Internal REST service | Deployment + Service + ConfigMap + Secret + probes |
| 02 | Public API | Ingress + TLS + Service + Deployment |
| 03 | REST + PostgreSQL + HPA | Secret + ConfigMap + DB endpoint + HPA + NetworkPolicy |
| 04 | Scheduled batch | CronJob + Job + ConfigMap + Secret + idempotency |
| 05 | File-processing worker | Deployment + PVC + ConfigMap + graceful shutdown |
| 06 | Service-to-service security | two Deployments + Services + NetworkPolicy + OAuth2 config |

## Как читать

Сначала откройте `README.md` конкретного стенда, затем `all.yaml`.

Обращайте внимание на mapping chains:

```text
Deployment.template.labels
      ↓
Service.spec.selector
      ↓
EndpointSlice
      ↓
Ready Pods
```

```text
containerPort.name=http
      ↓
Service.targetPort=http
      ↓
Pod:8080
```

```text
ConfigMap.data.CUSTOMER_API_URL
      ↓
envFrom
      ↓
Spring Environment
      ↓
${CUSTOMER_API_URL}
```

```text
resources.requests.cpu
      ↓
HPA CPU utilization denominator
```

## Важное ограничение

Это reference stands, а не universal production templates. Реальный cluster может отличаться:
- IngressClass;
- CNI;
- StorageClass;
- secret manager;
- observability stack;
- identity provider;
- PostgreSQL operator;
- organizational policies.

Поэтому каждый стенд явно разделяет **portable Kubernetes API** и **platform-specific assumptions**.
