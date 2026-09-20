# Prompt 04 — Deployments, Release Strategies, Observability and Day-2 Operations

Исследуй полный жизненный цикл Spring Boot Deployment.

## Deployment baseline

Разобрать каждое поле production manifest:
- replicas;
- selector/labels;
- container ports;
- image tag/digest;
- imagePullPolicy;
- resources requests/limits;
- probes;
- env/config mounts;
- securityContext;
- ServiceAccount;
- topologySpreadConstraints;
- affinity/anti-affinity;
- PDB;
- terminationGracePeriodSeconds;
- revisionHistoryLimit.

## Release strategies

Разделить:

### Native Kubernetes
- RollingUpdate;
- Recreate;
- rollout pause/resume;
- rollback;
- maxSurge;
- maxUnavailable.

### Progressive delivery
Исследовать как отдельный production layer:
- blue/green;
- canary;
- traffic splitting;
- metrics-driven promotion;
- rollback.

Объяснить, что Kubernetes Deployment сам по себе предоставляет RollingUpdate/Recreate, а полноценный canary/blue-green часто требует ingress/service manipulation или progressive-delivery controller.

## Spring Boot zero-downtime
Разобрать:
- readiness before traffic;
- startup probe;
- graceful shutdown;
- connection draining;
- in-flight requests;
- DB migration compatibility;
- backwards-compatible API/contracts;
- Kafka consumer shutdown/rebalancing.

## CI/CD / GitOps
Сравнить:
- imperative deploy from CI;
- Helm/Kustomize;
- GitOps controller model.

Зафиксировать разделение:
application source,
image artifact,
deployment configuration,
environment-specific values,
secrets.

## Observability
Минимум:
- logs to stdout/stderr;
- metrics;
- traces;
- Actuator;
- Prometheus/OpenTelemetry concepts;
- Kubernetes events;
- `kubectl describe`;
- `kubectl logs --previous`;
- rollout status/history.

## SRE scenarios
Практически разобрать:
- CrashLoopBackOff;
- ImagePullBackOff;
- OOMKilled;
- Pending;
- probe failure;
- no endpoints;
- DNS failure;
- bad rollout;
- dependency outage;
- node drain.

## Deliverables
- `docs/knowledge/07-deployment-release-operations.md`;
- `examples/spring-boot-deployment/`;
- `labs/07-rollout-debugging/`.
