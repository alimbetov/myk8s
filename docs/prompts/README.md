# Kubernetes Research Prompt Framework

Цель этой папки — не хранить короткие ответы, а задавать единый стандарт глубокого исследования и документирования Kubernetes для Spring Boot-разработчика и подготовки к CKAD.

## Как работать

Для каждой новой темы:

1. Выбрать специализированный prompt из этой папки.
2. Передать его вместе с конкретной темой/контекстом проекта.
3. Провести актуальное исследование по официальным источникам.
4. Сформировать учебный документ в `docs/knowledge/<NN>-<topic>.md`.
5. Добавить production-примеры в `examples/<topic>/`.
6. Добавить hands-on лабораторию в `labs/<topic>/`.
7. Отдельно отметить:
   - что обязан понимать Spring Boot developer;
   - что относится к обязанностям Kubernetes/platform administrator;
   - что входит в CKAD;
   - что важно в production, но не требуется на CKAD.
8. Пройти quality gate из master prompt.

## Главный принцип

Материал строится не вокруг YAML-объектов, а вокруг жизненного цикла приложения:

```text
Spring Boot
  -> external configuration
  -> container image
  -> Deployment/Pod
  -> Service/DNS
  -> security boundary
  -> PostgreSQL/Kafka/RabbitMQ/object storage
  -> persistent storage
  -> observability
  -> rollout/rollback
  -> backup/recovery
  -> day-2 operations
```

## Требуемая глубина

Каждая тема должна содержать:

- ментальную модель;
- production architecture;
- Spring Boot configuration;
- Kubernetes manifests;
- anti-patterns;
- security implications;
- troubleshooting;
- day-2 operations;
- CKAD mapping;
- лабораторную работу;
- вопросы для самопроверки.

## Приоритет источников

1. Kubernetes official documentation.
2. Spring official documentation.
3. CNCF / Linux Foundation certification curriculum.
4. Официальная документация продукта/оператора:
   PostgreSQL/CloudNativePG, Strimzi/Apache Kafka, RabbitMQ, storage operator/vendor.
5. Только затем — качественные secondary sources.

Все изменяемые во времени утверждения должны проверяться перед документированием.
