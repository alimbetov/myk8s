# Prompt 01 — Spring Boot Configuration, URLs and Secrets in Kubernetes

Исследуй и задокументируй полный configuration contract Spring Boot приложения в Kubernetes.

## Обязательные вопросы

### Spring configuration
Разобрать:
- `application.yaml` vs `application.properties`;
- profiles;
- property precedence;
- environment variables;
- `@ConfigurationProperties` vs массовый `@Value`;
- placeholders вида `${DB_HOST:localhost}`;
- config tree;
- mounted configuration files;
- refresh/restart semantics при изменении ConfigMap/Secret.

### Dependency URLs
Показать правильные варианты для:
- PostgreSQL JDBC;
- Kafka bootstrap servers;
- RabbitMQ;
- Redis;
- internal HTTP service;
- external HTTP API;
- S3-compatible storage.

Объяснить Kubernetes DNS:
`service`,
`service.namespace`,
`service.namespace.svc.cluster.local`.

Не использовать IP Pod как application dependency.

### Secret handling
Сравнить:
1. plain Kubernetes Secret;
2. Secret mounted as environment variables;
3. Secret mounted as files/config tree;
4. external secret manager + CSI/External Secrets-подход;
5. GitOps encrypted secrets (описать варианты и trade-offs).

Обязательно зафиксировать:
- base64 не является encryption;
- encryption at rest для etcd;
- RBAC least privilege;
- namespace isolation;
- secret rotation;
- avoiding secrets in logs;
- avoiding secrets in command-line arguments;
- avoiding plaintext secrets in Git.

### Spring Boot production example
Построить пример с typed config:

```java
@ConfigurationProperties(prefix = "app")
public record AppProperties(...) {}
```

и `application.yaml`, где адреса и tuning defaults могут жить в ConfigMap, а credentials — в Secret.

### Health and lifecycle
Разобрать:
- liveness;
- readiness;
- startup probe;
- Spring Boot Actuator probes;
- graceful shutdown;
- `terminationGracePeriodSeconds`;
- PreStop только если действительно требуется.

### Deliverables
Создать:
- `docs/knowledge/01-spring-boot-configuration.md`;
- `examples/spring-boot-config/`;
- `labs/01-config-secrets/`.

Добавить таблицу:
`property -> source -> confidential? -> owner -> rotation/reload behavior`.
