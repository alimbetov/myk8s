# Kubernetes Production Knowledge Library

Проверено: 2026-09-20.

Эта библиотека превращает prompt framework проекта `myk8s` в практическую production-grade базу знаний для Spring Boot-разработчика.

## Version baseline

- Kubernetes docs baseline: **v1.37** (latest minor released 2026-08-26).
- CKAD exam baseline: **Kubernetes v1.35** на дату проверки.
- Spring Boot reference baseline: **4.1.x**; большинство принципов применимо и к Boot 3.x.
- Для лабораторий допускается k3s, но различия дистрибутива должны быть отмечены отдельно.

## Карта библиотеки

1. [Platform baseline](00-platform-baseline.md)
2. [Spring Boot configuration and secrets](01-spring-boot-configuration-secrets.md)
3. [Services, DNS and service-to-service security](02-services-dns-service-security.md)
4. [Deployments, probes, resources and rollouts](03-deployments-probes-rollouts.md)
5. [Stateful dependencies](04-stateful-dependencies.md)
6. [Day-2 operations and failure playbook](05-day2-failure-playbook.md)
7. [CKAD production mapping](06-ckad-map.md)

Практика: [labs/](../../labs/)  
Reusable manifests: [examples/](../../examples/)

## Правило чтения

Каждую тему рассматриваем одновременно через четыре слоя:

```text
Spring Boot process
        |
        v
Pod / workload controller
        |
        v
Service / DNS / storage / network
        |
        v
Platform operations + security + observability
```

Network isolation не заменяет authentication. Secret не является полноценным secret-management lifecycle. Readiness не равна liveness. StatefulSet не превращает базу данных в HA-систему.

## Источники

- Kubernetes releases: https://kubernetes.io/releases/
- Kubernetes concepts: https://kubernetes.io/docs/concepts/
- Spring Boot reference: https://docs.spring.io/spring-boot/reference/
- CKAD: https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/

Дата проверки источников: **2026-09-20**.
