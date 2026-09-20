# myk8s Roadmap

## Phase 0 — Foundation and mental model
- cluster/control plane/node;
- Pod lifecycle;
- declarative desired state;
- labels/selectors;
- namespaces;
- kubectl/debugging baseline.

## Phase 1 — Spring Boot application contract
- externalized configuration;
- URLs and Kubernetes DNS;
- ConfigMap;
- Secret;
- config tree;
- probes;
- graceful shutdown;
- resource requests/limits.

## Phase 2 — Internal networking and security
- Service/EndpointSlice/CoreDNS;
- ingress vs internal traffic;
- NetworkPolicy;
- ServiceAccount/RBAC;
- Pod security;
- JWT/OAuth2/mTLS and trust boundaries.

## Phase 3 — Stateful dependencies
- PostgreSQL;
- Kafka;
- RabbitMQ;
- object/file storage;
- StorageClass/PV/PVC;
- Operators;
- backup/recovery;
- HA/DR.

## Phase 4 — Delivery
- Deployment;
- RollingUpdate/Recreate;
- rollback;
- Kustomize/Helm;
- CI/CD;
- GitOps;
- progressive delivery.

## Phase 5 — Operations
- logs/metrics/traces;
- capacity;
- autoscaling;
- PDB;
- drain/maintenance;
- incidents;
- backup restore drills;
- credential/certificate rotation.

## Phase 6 — CKAD intensive
- exam-domain labs;
- timed drills;
- weak-area repetitions;
- full mock scenarios.

## Documentation rule

Каждая phase сначала исследуется через prompt из `docs/prompts/`, затем превращается в:
- knowledge document;
- example manifests;
- lab;
- operational checklist;
- CKAD mapping where applicable.
