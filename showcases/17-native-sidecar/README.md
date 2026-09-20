# Showcase 17 — Native Kubernetes sidecar

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Место этого стенда в общей системе

```text
Pod
 |
 +--> Spring Boot main process
 |
 +--> Sidecar helper process
       |
       +--> telemetry/proxy/agent
```

Sidecar находится **внутри той же workload unit**, поэтому масштабируется и заменяется вместе с приложением.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

Проверено: 2026-09-20.

## Схема

\`\`\`text
Pod
 |
 +--> native sidecar telemetry-agent
 |       |
 |       +--> starts before app
 |       +--> restartPolicy: Always
 |       +--> continues for Pod lifetime
 |
 +--> Spring Boot app
\`\`\`

## Современный sidecar lifecycle

Kubernetes native sidecars реализуются как entries в:

\`\`\`yaml
initContainers:
  - restartPolicy: Always
\`\`\`

В отличие от обычного initContainer, такой container **не завершается перед main container**, а продолжает работать рядом с ним.

Native sidecars stable с Kubernetes 1.33.

## Почему это лучше обычных two app containers

Обычные containers в \`containers:\` запускаются без sidecar ordering contract.

Native sidecar даёт lifecycle semantics:
- starts during init sequence;
- может иметь startup/readiness probes;
- остается running;
- shutdown ordering учитывает sidecar role.

## Example use case

В стенде sidecar представляет локальный telemetry agent.

Spring application отправляет telemetry на:

\`\`\`text
http://127.0.0.1:4318
\`\`\`

Поскольку containers одного Pod разделяют network namespace, localhost общий.

## Mapping

\`\`\`text
app OTEL_EXPORTER_OTLP_ENDPOINT
        ↓
http://127.0.0.1:4318
        ↓
sidecar containerPort 4318
\`\`\`

Service не нужен для communication внутри одного Pod.

## Resource accounting

Sidecar потребляет CPU/memory того же Pod/node.

Нельзя считать его «бесплатным».

Sizing должен учитывать:
- app;
- sidecar;
- startup init containers.

## Availability coupling

Если sidecar обязателен для readiness/security, его failure должен отражаться в Pod readiness policy.

Если telemetry best-effort, нельзя превращать outage collector в outage business API без причины.

## Failure simulations

1. Sidecar image pull fails -> Pod startup affected.
2. Sidecar port wrong -> app telemetry fails, business API может продолжить.
3. sidecar CPU limit слишком мал -> telemetry backlog.
4. app uses \`localhost\` expecting another Pod -> показать, что localhost работает только внутри этого Pod.
5. sidecar config invalid -> restart independently of app.

## Проверка

\`\`\`bash
kubectl get pod
kubectl describe pod <pod>
kubectl logs <pod> -c telemetry-agent
kubectl logs <pod> -c app
\`\`\`

## Sources

- https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/
