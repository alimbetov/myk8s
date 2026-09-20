# Production knowledge library — expansion roadmap

Проверено: 2026-09-20.

Цель — последовательно превратить каждый research prompt в законченный production module по quality gate из master prompt.

## Wave 1 — Spring Boot workload foundation

- [x] platform mental model
- [x] ConfigMap/Secret + Spring external config
- [x] Service/DNS basics
- [x] probes/resources/rollout baseline
- [x] first troubleshooting lab
- [ ] container image and JVM ergonomics
- [ ] requests/limits/GC/native memory deep dive
- [ ] graceful shutdown for HTTP, Kafka and Rabbit consumers
- [ ] Jobs/CronJobs for Spring Batch
- [ ] autoscaling: HPA/VPA/KEDA trade-offs

## Wave 2 — Networking and security

- [ ] ingress vs Gateway API
- [ ] TLS termination patterns
- [ ] OAuth2/JWT resource server in Kubernetes
- [ ] workload identity and mTLS
- [ ] ServiceAccount/RBAC deep dive
- [ ] Pod Security Standards / SecurityContext / seccomp
- [ ] default-deny ingress+egress policy with DNS
- [ ] egress control and external dependencies
- [ ] secret managers / CSI / rotation runbooks

## Wave 3 — Stateful platform

- [ ] PostgreSQL: managed vs CloudNativePG vs manual
- [ ] PostgreSQL backup/restore/PITR lab
- [ ] Kafka with Strimzi
- [ ] RabbitMQ Cluster Operator
- [ ] object storage: S3-compatible/RustFS/MinIO patterns
- [ ] PVC/PV/StorageClass/CSI deep dive
- [ ] capacity, PDB, affinity and topology
- [ ] RPO/RTO and disaster-recovery drills

## Wave 4 — Delivery and operations

- [ ] RollingUpdate deep dive
- [ ] blue/green
- [ ] canary
- [ ] Helm
- [ ] Kustomize
- [ ] GitOps
- [ ] schema migration expand/contract
- [ ] observability: metrics/logs/traces
- [ ] SLO/SLI/error budget
- [ ] incident runbooks
- [ ] node drain/storage failure/full disk labs

## Wave 5 — CKAD track

CKAD current published domains:
- Application Design and Build — 20%
- Application Deployment — 20%
- Application Observability and Maintenance — 15%
- Application Environment, Configuration and Security — 25%
- Services and Networking — 20%

Для каждой production topic создается отдельный exam-speed exercise.

## Definition of done for each module

Модуль не считается законченным без:
- mental model;
- Spring Boot example;
- minimal + production Kubernetes manifests;
- security;
- operations;
- failure modes;
- troubleshooting;
- anti-patterns;
- developer/platform responsibility matrix;
- CKAD mapping;
- hands-on lab;
- official sources and verification date.
