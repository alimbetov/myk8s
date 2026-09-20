# Kubernetes + Spring Boot glossary

Короткий словарь нужен, чтобы новичок не останавливался на каждом термине.

| Термин | Простое объяснение |
|---|---|
| Desired state | Какое состояние мы описали в Kubernetes и хотим постоянно поддерживать |
| Reconciliation | Controller сравнивает desired и actual state и пытается устранить разницу |
| Pod | Минимальная schedulable workload unit; один или несколько тесно связанных containers |
| Deployment | Controller для stateless replicated Pods и rollout |
| ReplicaSet | Поддерживает нужное число Pods для Deployment revision |
| Service | Стабильная network identity перед изменяющимися Pods |
| EndpointSlice | Актуальный список backend endpoints Service |
| Ready | Pod можно использовать для обычного traffic |
| Running | Container process запущен; это не гарантирует Ready |
| ConfigMap | Non-secret runtime configuration |
| Secret | Kubernetes object для sensitive values; не полный secret lifecycle |
| PVC | Запрос workload на persistent storage |
| StorageClass | Правила/driver provisioning storage |
| StatefulSet | Stable identity/storage/order для stateful workloads |
| Operator | Controller, знающий domain semantics конкретного продукта |
| Ingress | HTTP(S) routing configuration перед Service |
| Gateway API | Более структурированная модель edge routing: GatewayClass/Gateway/Route |
| NetworkPolicy | L3/L4 network reachability policy |
| ServiceAccount | Workload identity для Kubernetes API |
| RBAC | Authorization к Kubernetes API |
| requests | Ресурсы, учитываемые scheduler; CPU request также важен для HPA utilization |
| limits | Runtime ceiling ресурсов |
| startupProbe | Даёт приложению время запуститься |
| livenessProbe | Решает, требуется ли restart container |
| readinessProbe | Решает, можно ли отправлять новый traffic |
| HPA | Меняет replica count по metrics |
| Job | Workload, который должен успешно завершиться |
| CronJob | Создаёт Jobs по расписанию |
| Headless Service | Service без обычного virtual ClusterIP; часто для stable peer discovery |
| RPO | Сколько данных допустимо потерять |
| RTO | За какое время нужно восстановить сервис |
| PITR | Point-in-time recovery |
| HikariCP | Connection pool, обычно используемый Spring Boot для JDBC |
| Consumer lag | Насколько Kafka consumer отстаёт от produced records |
| Quorum | Минимальное согласованное большинство участников distributed system |
| Sidecar | Helper container, живущий в одном Pod lifecycle с main workload |
| initContainer | Container, выполняющий подготовку до main containers |

## Как использовать словарь

Если термин впервые появляется в главе, сначала получите **простую mental model**, а затем возвращайтесь к точному API definition.

Учебник сознательно использует Kubernetes/Spring jargon, потому что его нужно узнавать на работе, но каждый термин должен быть привязан к concrete runtime behavior.
