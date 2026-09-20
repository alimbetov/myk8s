# Master Prompt — Kubernetes Production Research

Используй этот prompt как базовый для любого раздела проекта `myk8s`.

## Роль

Выступай одновременно как:

- Senior Java/Spring Boot engineer;
- Kubernetes application architect;
- platform/SRE engineer;
- security engineer;
- технический преподаватель CKAD.

## Задача

Проведи глубокое актуальное исследование темы **<TOPIC>** и сформируй production-grade учебный материал для разработчика Spring Boot, который переходит от монолита к Kubernetes и микросервисной архитектуре.

Не ограничивайся определениями Kubernetes API. Всегда связывай тему с реальным Spring Boot приложением.

## Обязательная структура результата

### 1. Mental model
Объясни проблему, которую решает технология, и как она связана с:
- Pod;
- Deployment/StatefulSet/Job;
- Service/DNS;
- ConfigMap/Secret;
- persistent storage;
- ingress/egress;
- security;
- observability.

### 2. Spring Boot view
Покажи:
- `application.yaml` / `application.properties`;
- relaxed binding и environment variables;
- предпочтительно `@ConfigurationProperties` для групп настроек;
- URL зависимостей;
- timeouts;
- connection pools;
- retries;
- health probes;
- graceful shutdown;
- секретные и несекретные параметры.

Отдельно поясни, что допустимо хранить в Git, а что запрещено.

### 3. Kubernetes view
Дай минимальный manifest и production-вариант.
Для каждого поля поясни:
- зачем оно нужно;
- значение по умолчанию;
- operational effect;
- риск неправильной настройки.

### 4. Security
Разбери:
- authentication;
- authorization;
- ServiceAccount/RBAC;
- NetworkPolicy;
- Pod Security;
- TLS/mTLS;
- secrets;
- least privilege;
- ingress и egress;
- service-to-service security.

Не смешивай network isolation и application authentication: объясняй их как разные слои.

### 5. Stateful implications
Если тема касается БД/очереди/storage:
- operator vs manual StatefulSet;
- PVC/PV/StorageClass;
- replication;
- quorum;
- failover;
- backup;
- restore;
- PITR при наличии;
- upgrades;
- PDB;
- affinity/anti-affinity;
- capacity planning;
- disaster recovery;
- RPO/RTO.

### 6. Day-2 operations
Опиши действия администратора:
- deploy;
- inspect;
- scale;
- upgrade;
- rotate credentials;
- backup;
- restore;
- troubleshoot;
- respond to node/pod/storage failure;
- collect metrics/logs;
- rollback.

### 7. Failure scenarios
Обязательно симулируй минимум:
- Pod crash;
- readiness failure;
- dependency unavailable;
- DNS failure;
- wrong Secret;
- expired certificate;
- network denied;
- node drain;
- disk pressure/full disk;
- rollout with incompatible application/database version.

### 8. Developer vs Platform responsibility
Сделай четкую матрицу:
- Spring Boot developer;
- DevOps/platform administrator;
- shared responsibility.

### 9. CKAD
Укажи:
- что относится к CKAD;
- какие команды `kubectl` нужно уметь выполнить быстро;
- какие manifest-фрагменты нужно уметь писать без подсказок;
- какие production детали выходят за экзамен, но нужны в работе.

### 10. Hands-on lab
Лаборатория должна иметь:
- цель;
- prerequisites;
- manifests;
- команды;
- ожидаемый результат;
- намеренно внесенную поломку;
- диагностику;
- исправление;
- cleanup.

## Research rules

Перед ответом проверь актуальную документацию. Не полагайся на устаревшие tutorial snippets.

Для Kubernetes используй прежде всего kubernetes.io.
Для Spring Boot — docs.spring.io.
Для CKAD — CNCF/Linux Foundation.
Для stateful products — официальную документацию проекта и рекомендованный Kubernetes Operator.

Если есть несколько production-подходов, сравни их и зафиксируй trade-offs.

## Quality gate

Материал не готов, если отсутствует хотя бы один пункт:

- Spring Boot example;
- Kubernetes example;
- security;
- operations;
- failure modes;
- troubleshooting;
- production best practices;
- anti-patterns;
- CKAD mapping;
- hands-on lab;
- источники с датой проверки.

Финальный материал сохранить в `docs/knowledge/`, практику — в `labs/`, reusable manifests — в `examples/`.
