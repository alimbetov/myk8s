# myk8s

Production-oriented Kubernetes knowledge base для Spring Boot-разработчика: **учебник + cookbook + CKAD workbook + production runbook**.

Цель проекта: по этому репозиторию можно с нуля понять Kubernetes как Java/Spring Boot разработчик, подготовиться к CKAD, развернуть реальное приложение и через полгода открыть тот же репозиторий во время production-инцидента.

## Начать здесь

- [Knowledge library index](docs/knowledge/README.md)
- [Documentation conventions](docs/knowledge/CONVENTIONS.md)
- [Production library roadmap](docs/knowledge/library-roadmap.md)
- [Master research prompt](docs/prompts/00-master-k8s-research.md)

## Что уже покрыто

### Foundation
- platform mental model;
- Spring Boot external configuration;
- ConfigMap/Secret;
- Services/DNS;
- Deployment/ReplicaSet/Pod;
- stateful overview;
- Day-2 troubleshooting;
- CKAD map.

### Spring Boot workload
- container images and JVM;
- CPU/memory requests and limits;
- JVM memory budget;
- probes and lifecycle;
- graceful shutdown;
- HTTP client DNS/timeouts/retries/pools;
- PostgreSQL connectivity and HikariCP.

### Networking and security
- Kubernetes service discovery;
- Eureka migration considerations;
- service-to-service authentication;
- OAuth2/JWT/mTLS layers;
- NetworkPolicy and egress;
- ServiceAccount/RBAC;
- secrets lifecycle/rotation;
- Ingress/Gateway API/TLS.

### Delivery and operations
- RollingUpdate/Recreate/blue-green/canary;
- HPA/autoscaling;
- Jobs/CronJobs/Spring Batch;
- observability.

## Architecture showcases

- [Production-like service stands](showcases/README.md)
- [Manifest mapping cheat sheet](showcases/MAPPING-CHEATSHEET.md)

Showcases демонстрируют не отдельный YAML object, а полную композицию нескольких ресурсов: Deployment, Service, ConfigMap, Secret, NetworkPolicy, HPA, Ingress, PVC и CronJob.

## Hands-on labs

- [Lab 01 — Spring Boot baseline](labs/01-spring-boot-baseline/README.md)
- [Lab 02 — JVM/resources/probes](labs/02-jvm-resources-probes/README.md)
- [Lab 03 — networking/security](labs/03-networking-security/README.md)
- [Lab 04 — rollout/HPA](labs/04-rollout-hpa/README.md)
- [Lab 05 — Jobs/operations](labs/05-jobs-operations/README.md)

## Reusable manifests

See [examples/](examples/):
- hardened Spring Boot Deployment;
- Service;
- ConfigMap/Secret template;
- ServiceAccount;
- NetworkPolicy;
- PDB;
- HPA;
- CronJob;
- least-privilege RBAC examples.

## Version baseline

Проверено 2026-09-20:
- production documentation: Kubernetes 1.37;
- CKAD published environment: Kubernetes 1.35;
- Spring Boot reference baseline: 4.1.x.

Современные возможности помечаются отдельно. Например, in-place container CPU/memory resize стабилен с Kubernetes 1.35; базовое обучение всё равно начинается с классической модели requests/limits.

## Метод

```text
problem
  -> mental model
  -> как было раньше
  -> как сейчас
  -> Spring Boot contract
  -> Kubernetes manifest
  -> runtime sequence
  -> security
  -> operations
  -> intentional failure
  -> diagnosis
  -> repair
  -> anti-patterns
  -> CKAD speed practice
```

Sources в конце глав предназначены для проверки и углубления. Основное объяснение должно находиться внутри самого `myk8s`.
