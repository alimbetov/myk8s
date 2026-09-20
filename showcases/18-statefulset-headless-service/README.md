# Showcase 18 — StatefulSet + Headless Service

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

Проверено: 2026-09-20.

## Схема

\`\`\`text
Headless Service: cluster-member
        |
        +--> cluster-member-0.cluster-member
        +--> cluster-member-1.cluster-member
        +--> cluster-member-2.cluster-member

StatefulSet
        |
        +--> Pod ordinal 0 -> PVC data-cluster-member-0
        +--> Pod ordinal 1 -> PVC data-cluster-member-1
        +--> Pod ordinal 2 -> PVC data-cluster-member-2
\`\`\`

## Когда StatefulSet нужен

Когда workload требует:
- stable unique network identity;
- stable persistent storage;
- ordered lifecycle;
- predictable ordinal identity.

Обычный stateless Spring Boot REST API обычно лучше Deployment.

## Headless Service

\`\`\`yaml
clusterIP: None
\`\`\`

Нет обычного virtual Service IP load balancing.

DNS предоставляет identities отдельных Pods.

## StatefulSet.serviceName

\`\`\`yaml
serviceName: cluster-member
\`\`\`

должно ссылаться на governing headless Service.

Pod names:

\`\`\`text
cluster-member-0
cluster-member-1
cluster-member-2
\`\`\`

DNS:

\`\`\`text
cluster-member-0.cluster-member
cluster-member-1.cluster-member
cluster-member-2.cluster-member
\`\`\`

в том же namespace.

## volumeClaimTemplates

Template:

\`\`\`yaml
metadata:
  name: data
\`\`\`

должен совпасть с:

\`\`\`yaml
volumeMounts:
  - name: data
\`\`\`

StatefulSet создаёт отдельный PVC для каждого ordinal.

## PVC lifecycle

По умолчанию StatefulSet-created PVCs сохраняются при scale down/delete, чтобы не потерять data автоматически.

Это data-safety default, но cleanup должен быть explicit runbook.

## podManagementPolicy

Default:

\`\`\`text
OrderedReady
\`\`\`

Pods scale/create ordered by ordinal.

\`Parallel\` можно использовать, если distributed product не требует startup ordering.

## StatefulSet != database HA

StatefulSet даёт identity/storage/order.

Он **не реализует**:
- leader election;
- replication protocol;
- quorum;
- failover;
- backup;
- consistency.

Поэтому PostgreSQL/Kafka/RabbitMQ лучше показывать через product-aware operator.

## Failure simulations

1. Delete \`cluster-member-1\` -> replacement получает тот же ordinal identity и свой PVC.
2. Headless Service name не совпадает с \`serviceName\` -> stable service discovery broken.
3. PVC Pending -> corresponding Pod startup blocked.
4. Pod-0 never Ready with OrderedReady -> later Pods may not progress as expected.
5. Scale down -> Pods уходят, PVCs остаются by default.

## Диагностика

\`\`\`bash
kubectl get statefulset
kubectl get pod
kubectl get svc cluster-member
kubectl get endpointslice
kubectl get pvc
kubectl describe statefulset cluster-member
\`\`\`

## Sources

- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- https://kubernetes.io/docs/reference/kubernetes-api/apps-workload-resources/stateful-set-v1/
