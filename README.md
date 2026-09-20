# myk8s

Практический проект по Kubernetes для Spring Boot-разработчика: архитектура, эксплуатация, безопасность и подготовка к CKAD.

## Текущий этап

Ветка `docs/k8s-research-prompts` формирует исследовательский каркас. На следующих итерациях каждый prompt превращается в полноценный knowledge document, runnable manifests и hands-on labs.

## Навигация

- [Roadmap](docs/roadmap.md)
- [Prompt framework](docs/prompts/README.md)
- [Master research prompt](docs/prompts/00-master-k8s-research.md)
- [Spring Boot configuration and secrets](docs/prompts/01-spring-boot-config-secrets.md)
- [Internal services, networking and security](docs/prompts/02-networking-service-security.md)
- [PostgreSQL, Kafka, RabbitMQ and storage](docs/prompts/03-stateful-platform.md)
- [Deployment, delivery and operations](docs/prompts/04-deployment-delivery-operations.md)
- [CKAD practice track](docs/prompts/05-ckad-practice.md)

## Метод

Каждая тема проходит одинаковый цикл:

```text
research
  -> mental model
  -> Spring Boot configuration
  -> Kubernetes manifests
  -> security
  -> production operations
  -> failure scenarios
  -> troubleshooting
  -> CKAD mapping
  -> hands-on lab
```

Production и экзаменационная подготовка связаны, но документируются раздельно: CKAD учит быстро и правильно работать с Kubernetes application primitives, а production-трек дополнительно охватывает operators, HA/DR, secrets lifecycle, stateful systems, observability и day-2 operations.
