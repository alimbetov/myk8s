# Kubernetes Manifest Coverage Matrix

Проверено: 2026-09-20.

Цель каталога — гарантировать, что каждый основной Kubernetes manifest проходит три отдельные итерации и не считается завершённым только потому, что где-то упомянут в тематической главе.

## Три итерации

### I1 — Structure pass
Сухой проход:
- назначение;
- apiVersion/kind;
- список важных metadata/spec fields;
- defaults, которые нужно раскрыть;
- runtime relations;
- troubleshooting commands;
- Spring Boot links;
- CKAD links.

### I2 — Full reference
Полный field-by-field разбор:
- зачем поле существует;
- тип/допустимые значения;
- default;
- runtime effect;
- связи с другими objects;
- минимальный manifest;
- production manifest;
- ошибки конфигурации;
- failure scenarios;
- диагностика;
- security/operations.

### I3 — Pedagogical refactor
Учебная переработка:
- простой язык;
- как было раньше / как сейчас;
- схемы процессов;
- пошаговые примеры;
- аналогии для Spring Boot разработчика;
- упражнения;
- anti-patterns;
- контрольные вопросы;
- быстрый incident/runbook режим.

## Coverage

| Manifest | I1 Structure | I2 Full | I3 Pedagogy | File |
|---|---:|---:|---:|---|
| Service | DONE | DONE | PARTIAL | [service.md](service.md) |
| Deployment | DONE | DONE | TODO | [deployment.md](deployment.md) |
| Pod | DONE | DONE | TODO | [pod.md](pod.md) |
| ConfigMap | DONE | DONE | TODO | [configmap.md](configmap.md) |
| Secret | DONE | DONE | TODO | [secret.md](secret.md) |
| NetworkPolicy | DONE | DONE | TODO | [networkpolicy.md](networkpolicy.md) |
| ServiceAccount + RBAC | DONE | DONE | TODO | [serviceaccount-rbac.md](serviceaccount-rbac.md) |
| PersistentVolumeClaim | DONE | DONE | TODO | [pvc.md](pvc.md) |
| Job + CronJob | DONE | DONE | TODO | [job-cronjob.md](job-cronjob.md) |
| HorizontalPodAutoscaler | DONE | DONE | TODO | [hpa.md](hpa.md) |
| Ingress | DONE | DONE | TODO | [ingress.md](ingress.md) |

## Iteration 1 status

**Core manifest coverage is complete for the requested set.**

На этом этапе каждый объект уже имеет отдельную страницу и skeleton полей. Это не означает, что reference завершён: подробная семантика и педагогическая переработка специально отложены на I2/I3, чтобы не смешивать стадии.

## Iteration 2 status

**Core manifest technical reference is complete for the requested set.**

На I2 для каждого объекта добавлены реальные defaults, field semantics, runtime/update behavior, immutable/mutable ограничения, связи с другими resources, security/Day-2, failure scenarios, troubleshooting и CKAD commands.

Следующая отдельная стадия — **I3 Pedagogical refactor**. В ней технический reference не переписывается заново, а перерабатывается для последовательного обучения: простой язык, схемы, «как было раньше / как сейчас», пошаговые labs и контрольные вопросы.

## Порядок I3

Рекомендуемый порядок полного разбора:

1. Deployment
2. Pod
3. ConfigMap
4. Secret
5. NetworkPolicy
6. ServiceAccount + Role/ClusterRole + bindings
7. PVC
8. Job
9. CronJob
10. HPA
11. Ingress

`Job/CronJob` и `ServiceAccount/RBAC` могут остаться в общих файлах, но внутри I2 каждый kind будет разобран отдельно.

## Definition of I2 FULL

Технический reference считается I2 FULL, если присутствуют:

- полный minimal example;
- production-oriented example;
- разбор каждого существенного поля;
- defaults;
- immutable/mutable behavior where relevant;
- creation/update/runtime sequence;
- relationship to other Kubernetes resources;
- Spring Boot usage;
- security considerations;
- Day-2 operations;
- at least 3 failure scenarios;
- troubleshooting algorithm;
- anti-patterns;
- CKAD commands;
- production checklist;
- official sources with verification date.

## Что пока намеренно не включено в core pass

Это будут следующие manifest tracks после core set:

- PodDisruptionBudget;
- StatefulSet;
- PersistentVolume;
- StorageClass;
- Gateway;
- HTTPRoute;
- Namespace;
- ResourceQuota;
- LimitRange;
- DaemonSet.

Они уже упоминаются в тематических главах, но пройдут тот же I1 → I2 → I3 pipeline отдельно.
