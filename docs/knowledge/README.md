# Kubernetes Production Knowledge Library

## Начать обучение

- [Как учиться по myk8s: аналитик, разработчик, тестировщик](LEARNING-GUIDE.md)
- [Kubernetes + Spring Boot glossary](GLOSSARY.md)
- [Documentation conventions](CONVENTIONS.md)

Если вы впервые открыли проект, начните с `LEARNING-GUIDE.md`, затем переходите к главе 00.

## Как теперь устроен учебник

Каждая основная тема читается в одном направлении:

```text
общая архитектурная схема
        ↓
роль конкретного механизма
        ↓
object relationship diagram
        ↓
runtime sequence
        ↓
annotated example
        ↓
technical reference
        ↓
production-like showcase
        ↓
failure / diagnostics
```

Для hands-on примеров используйте два YAML-варианта:

- `showcases/*/annotated.yaml` — учебный manifest с комментариями;
- `showcases/*/all.yaml` — чистый apply-ready manifest.

Так документация одновременно остаётся:
- учебником;
- Kubernetes API reference;
- Spring Boot cookbook;
- production incident runbook.

Проверено: 2026-09-20.

Эта библиотека — учебник, Spring Boot cookbook, CKAD workbook и production runbook в одном репозитории.

## Version baseline

- Production research: Kubernetes **1.37**.
- CKAD exam baseline на дату проверки: Kubernetes **1.35**.
- Spring Boot reference baseline: **4.1.x**; принципы применимы и к Boot 3.x.
- Labs можно выполнять на k3s с учетом особенностей bundled CNI/Ingress/StorageClass.

## Стандарт документации

- [Documentation conventions](CONVENTIONS.md)

Каждая глава должна быть понятна без обязательного перехода во внешние Sources. Ссылки используются для проверки спецификации и углубления.

## Foundation

1. [00 — Platform baseline](00-platform-baseline.md)
2. [01 — Spring Boot configuration and secrets](01-spring-boot-configuration-secrets.md)
3. [02 — Services, DNS and service-to-service security](02-services-dns-service-security.md)
4. [03 — Deployments, probes, resources and rollouts](03-deployments-probes-rollouts.md)
5. [04 — Stateful dependencies](04-stateful-dependencies.md)
6. [05 — Day-2 operations and failure playbook](05-day2-failure-playbook.md)
7. [06 — CKAD map](06-ckad-map.md)

## Spring Boot workload track

8. [07 — Container image + JVM in Kubernetes](07-container-image-jvm.md)
9. [08 — Resources, JVM memory and CPU](08-resources-jvm-memory-cpu.md)
10. [09 — Spring Boot probes and lifecycle](09-spring-boot-probes-lifecycle.md)
11. [10 — Graceful shutdown and termination](10-graceful-shutdown-termination.md)
12. [11 — HTTP clients: DNS, timeout, retry and pool](11-http-clients-dns-timeout-retry-pool.md)
13. [12 — PostgreSQL connectivity and HikariCP](12-postgresql-connectivity-hikaricp.md)

## Networking and security track

14. [13 — Service discovery and internal services](13-service-discovery-internal-services.md)
15. [14 — Service-to-service authentication](14-service-to-service-authentication.md)
16. [15 — NetworkPolicy and egress](15-networkpolicy-egress.md)
17. [16 — ServiceAccount and RBAC](16-serviceaccount-rbac.md)
18. [17 — Secrets lifecycle and rotation](17-secrets-lifecycle-rotation.md)
19. [18 — Ingress, Gateway API and TLS](18-ingress-gateway-api-tls.md)

## Delivery and operations track

20. [19 — Deployment strategies](19-deployment-strategies.md)
21. [20 — HPA / autoscaling](20-hpa-autoscaling.md)
22. [21 — Jobs / CronJobs / Spring Batch](21-jobs-cronjobs-spring-batch.md)
23. [22 — Observability](22-observability.md)

## Kubernetes manifest reference

- [Manifest coverage matrix](manifests/README.md)
- [Service — полный разбор manifest](manifests/service.md)

Этот раздел отвечает именно на вопрос **«что означает каждое поле YAML, какой у него default, runtime effect и как его диагностировать»**. Тематические главы объясняют архитектуру; manifest reference — конкретный Kubernetes API object построчно.

## Ознакомительные architecture showcases

- [Showcase catalog](../../showcases/README.md)
- [Manifest mapping cheat sheet](../../showcases/MAPPING-CHEATSHEET.md)

Эти стенды показывают совместную работу нескольких manifests на одном service scenario: mapping labels/selectors/ports/config/secrets, runtime traffic, autoscaling, storage и security boundaries.

## Практика

- [Lab 01 — Spring Boot Kubernetes baseline](../../labs/01-spring-boot-baseline/README.md)
- [Lab 02 — JVM/resources/probes](../../labs/02-jvm-resources-probes/README.md)
- [Lab 03 — networking/security](../../labs/03-networking-security/README.md)
- [Lab 04 — rollout/HPA](../../labs/04-rollout-hpa/README.md)
- [Lab 05 — Jobs/CronJobs/operations](../../labs/05-jobs-operations/README.md)

Reusable manifests: [examples/](../../examples/)

## Как пользоваться библиотекой

Для обучения:

```text
прочитать mental model
 -> повторить YAML/Java
 -> выполнить commands
 -> сломать scenario
 -> диагностировать
 -> исправить
 -> повторить CKAD speed round
```

Во время incident:

```text
symptom
 -> 05 Day-2 playbook
 -> тематическая глава
 -> failure practice/runbook
 -> verify recovery
```

Главный принцип: Kubernetes изучается через эксплуатационный контракт реального Spring Boot приложения, а не как набор YAML API.


## I3 quality coverage

Редакторский I3-pass проверяет не только техническую полноту, но и обучаемость.

| Диапазон | Учебная карта | Роли analyst/dev/tester | Runtime/failure thinking | Production-like examples |
|---|---:|---:|---:|---:|
| 00–06 foundation | DONE | DONE | DONE | DONE |
| 07–12 workload/runtime | DONE | DONE | DONE | DONE |
| 13–18 networking/security/edge | DONE | DONE | DONE | DONE |
| 19–22 delivery/operations | DONE | DONE | DONE | DONE |

### Что означает DONE

Глава должна позволять ответить без заучивания YAML:

```text
Почему механизм существует?
Где он находится в общей архитектуре?
Кто его исполняет?
С чем он связан?
Какими полями связаны objects?
Что происходит runtime?
Как failure проявится в Spring Boot?
Как это проверить?
Что важно аналитику?
Что важно разработчику?
Что важно тестировщику?
```

Следующий редакторский уровень — проход по тексту на локальные разрывы объяснения: undefined terminology, слишком резкие переходы, примеры без причинно-следственной связи и недостаточно подробные failure narratives.
