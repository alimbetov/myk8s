# Production knowledge library — roadmap

Проверено: 2026-09-20.

## Wave 1 — Foundation — DONE

- [x] platform mental model
- [x] configuration and secrets baseline
- [x] Services/DNS/security baseline
- [x] Deployments/probes/rollouts
- [x] stateful overview
- [x] Day-2 troubleshooting
- [x] CKAD map

## Wave 2 — Spring Boot workload foundation — DONE

- [x] container image + JVM
- [x] resources/JVM memory/CPU
- [x] probes and Spring lifecycle
- [x] graceful shutdown
- [x] HTTP clients/timeouts/retries/pools
- [x] PostgreSQL/Hikari connectivity

## Wave 3 — Networking and security foundation — DONE

- [x] Service discovery / internal services
- [x] service-to-service authentication
- [x] NetworkPolicy and egress
- [x] ServiceAccount/RBAC
- [x] secrets lifecycle/rotation
- [x] Ingress/Gateway API/TLS

## Wave 4 — Delivery and operations foundation — DONE

- [x] deployment strategies
- [x] HPA/autoscaling
- [x] Jobs/CronJobs/Spring Batch
- [x] observability

## Wave 5 — Deep stateful track — NEXT

Каждый продукт получает отдельный набор production chapters, labs и manifests.

### PostgreSQL
- [ ] PostgreSQL architecture in Kubernetes
- [ ] CloudNativePG
- [ ] replication/failover
- [ ] backup/restore/PITR
- [ ] storage/capacity
- [ ] upgrades
- [ ] disaster recovery
- [ ] troubleshooting lab

### Kafka
- [ ] Kafka architecture refresher
- [ ] Strimzi
- [ ] KRaft/controller quorum
- [ ] replication/ISR
- [ ] storage/capacity
- [ ] upgrades
- [ ] security
- [ ] failure lab

### RabbitMQ
- [ ] RabbitMQ architecture refresher
- [ ] Cluster Operator
- [ ] quorum queues
- [ ] disk/memory alarms
- [ ] persistence
- [ ] upgrades
- [ ] security
- [ ] failure lab

### Object storage
- [ ] S3 model
- [ ] MinIO/RustFS patterns
- [ ] Spring Boot client
- [ ] presigned upload/download
- [ ] retention
- [ ] backup/DR

## Wave 6 — Platform engineering deep dive

- [ ] Helm
- [ ] Kustomize
- [ ] GitOps
- [ ] namespaces/quotas/LimitRange
- [ ] scheduling/taints/tolerations
- [ ] affinity/topology spread
- [ ] PDB deep dive
- [ ] VPA/KEDA
- [ ] Pod Security Standards
- [ ] policy engines
- [ ] multi-cluster/DR

## Definition of done

Модуль считается законченным, когда содержит:
- mental model;
- как было раньше / как сейчас;
- Spring Boot example;
- minimal + production Kubernetes example;
- почему выбран каждый важный parameter;
- defaults;
- security;
- operations;
- failure scenarios;
- troubleshooting;
- anti-patterns;
- responsibility matrix;
- CKAD mapping;
- hands-on practice;
- official sources с датой проверки.
