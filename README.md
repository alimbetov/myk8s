# myk8s

Production-oriented Kubernetes knowledge base для Spring Boot-разработчика: архитектура, эксплуатация, безопасность, stateful platform и подготовка к CKAD.

## Что уже есть

### Research framework
- [Master research prompt](docs/prompts/00-master-k8s-research.md)
- [Prompt framework](docs/prompts/README.md)
- [Research roadmap](docs/roadmap.md)

### Production knowledge library
- [Library index](docs/knowledge/README.md)
- [00 — Platform baseline](docs/knowledge/00-platform-baseline.md)
- [01 — Spring Boot configuration and secrets](docs/knowledge/01-spring-boot-configuration-secrets.md)
- [02 — Services, DNS and service-to-service security](docs/knowledge/02-services-dns-service-security.md)
- [03 — Deployments, probes, resources and rollouts](docs/knowledge/03-deployments-probes-rollouts.md)
- [04 — Stateful dependencies](docs/knowledge/04-stateful-dependencies.md)
- [05 — Day-2 operations and failure playbook](docs/knowledge/05-day2-failure-playbook.md)
- [06 — CKAD map](docs/knowledge/06-ckad-map.md)
- [Library expansion roadmap](docs/knowledge/library-roadmap.md)

### Hands-on
- [Lab 01 — Spring Boot Kubernetes baseline](labs/01-spring-boot-baseline/README.md)

### Reusable manifests
- [Spring Boot base manifests](examples/spring-boot/base/)

## Version baseline

Проверено 2026-09-20:
- production docs ориентируются на Kubernetes 1.37;
- CKAD exam page указывает Kubernetes 1.35;
- Spring Boot reference — 4.1.x;
- лаборатории могут выполняться на k3s с обязательным учетом различий дистрибутива.

## Метод

```text
research
  -> mental model
  -> Spring Boot contract
  -> Kubernetes manifests
  -> security layers
  -> stateful implications
  -> day-2 operations
  -> failure injection
  -> troubleshooting
  -> developer/platform responsibility
  -> CKAD mapping
  -> hands-on lab
```

Главный принцип проекта: Kubernetes API изучается не изолированно, а через эксплуатационный контракт реального Spring Boot приложения.
