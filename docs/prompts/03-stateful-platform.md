# Prompt 03 — PostgreSQL, Kafka, RabbitMQ and File/Object Storage

Цель — научиться не просто «запускать контейнер с БД», а понимать production ownership stateful workloads.

## Общая модель

Для каждого продукта исследовать:

- managed external service vs run inside Kubernetes;
- recommended Kubernetes Operator;
- StatefulSet/CRD model;
- Service/headless Service;
- PVC/PV/StorageClass;
- topology and failure domains;
- replica placement;
- quorum/leader/primary;
- backup/restore;
- upgrade;
- monitoring;
- credentials/TLS;
- NetworkPolicy;
- resource sizing;
- disk growth;
- disaster recovery;
- admin runbook.

## PostgreSQL

Сравнить:
- external managed PostgreSQL;
- standalone StatefulSet (только lab/dev или осознанный special case);
- PostgreSQL Operator, отдельно исследовать CloudNativePG.

Разобрать:
- primary/replicas;
- synchronous vs asynchronous replication;
- failover/switchover;
- PgBouncer;
- WAL;
- physical backup;
- PITR;
- backup to S3-compatible object storage;
- restore test;
- schema migration ownership (Liquibase/Flyway) и zero-downtime migrations.

## Kafka

Исследовать Apache Kafka в Kubernetes через operator-подход, в первую очередь Strimzi:
- KRaft;
- brokers/controllers;
- listeners;
- internal/external access;
- topics/users as declarative resources where applicable;
- replication factor;
- partitions;
- PVC;
- rack/topology awareness;
- rolling upgrades;
- rebalancing;
- TLS/SASL;
- quotas;
- monitoring;
- backup/DR limitations и replication strategy.

## RabbitMQ

Исследовать официальный RabbitMQ Cluster Operator и Messaging Topology Operator:
- cluster provisioning;
- quorum queues;
- users/vhosts/exchanges/queues/bindings;
- TLS;
- credentials;
- PVC;
- PDB;
- NetworkPolicy;
- monitoring quorum;
- upgrade;
- failure/recovery.

## File/Object storage

Разделить:
- Kubernetes filesystem volume;
- shared filesystem;
- object storage S3-compatible;
- application uploads/artifacts;
- database backup target.

Исследовать operational варианты для production и отдельно локальную учебную установку.

## Spring Boot clients

Для каждого продукта показать корректный `application.yaml`:
- endpoint/DNS;
- credentials from Secret;
- TLS;
- timeouts;
- pool/thread settings;
- health behavior;
- retry policy;
- startup dependency anti-patterns.

## Admin runbook

Для каждого stateful компонента:
- daily checks;
- capacity;
- replication health;
- certificate expiration;
- backup success;
- restore drill;
- upgrades;
- incident response;
- node drain;
- PVC issue;
- storage full;
- credential rotation.

## Deliverables
- `docs/knowledge/03-postgresql.md`;
- `docs/knowledge/04-kafka.md`;
- `docs/knowledge/05-rabbitmq.md`;
- `docs/knowledge/06-storage.md`;
- соответствующие `examples/` и `labs/`.
