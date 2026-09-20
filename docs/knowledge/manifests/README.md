# Kubernetes Manifest Coverage Matrix

Проверено: 2026-09-20.

Этот каталог нужен, чтобы в библиотеке не было ситуации: manifest есть в `examples/`, но подробного объяснения его полей нет.

## Статусы

- **FULL** — есть отдельный field-by-field reference.
- **TOPIC** — механизм подробно раскрыт в тематической главе, но отдельный manifest reference ещё нужен.
- **TODO** — требуется отдельный справочник.

| Manifest | Example | Topic knowledge | Field-by-field reference | Status |
|---|---|---|---|---|
| Pod | indirectly via Deployment | 00, 03, 09 | TODO | TODO |
| Deployment | yes | 03, 19 | TODO | TOPIC |
| Service | yes | 02, 13 | [Service reference](service.md) | FULL |
| ConfigMap | yes | 01 | TODO | TOPIC |
| Secret | template | 01, 17 | TODO | TOPIC |
| ServiceAccount | yes | 16 | TODO | TOPIC |
| Role | yes | 16 | TODO | TOPIC |
| RoleBinding | yes | 16 | TODO | TOPIC |
| NetworkPolicy | yes | 15 | TODO | TOPIC |
| PodDisruptionBudget | yes | 03 | TODO | TOPIC |
| HPA | yes | 20 | TODO | TOPIC |
| Job | discussed | 21 | TODO | TOPIC |
| CronJob | yes | 21 | TODO | TOPIC |
| PVC | discussed | 04 | TODO | TOPIC |
| StatefulSet | discussed | 04 | TODO | TOPIC |
| Ingress | discussed | 18 | TODO | TOPIC |
| Gateway | discussed | 18 | TODO | TOPIC |
| HTTPRoute | discussed | 18 | TODO | TOPIC |

## Следующий порядок заполнения

### Core CKAD manifests

1. Deployment
2. Pod
3. ConfigMap
4. Secret
5. NetworkPolicy
6. ServiceAccount + Role + RoleBinding
7. PVC
8. Job
9. CronJob
10. HPA
11. Ingress

### Production extensions

12. PDB
13. StatefulSet
14. Gateway
15. HTTPRoute

## Definition of FULL

Manifest считается полноценно задокументированным только если описаны:

- `apiVersion`;
- `kind`;
- metadata;
- основные `spec` fields;
- defaults;
- связь с другими resources;
- runtime effect;
- Spring Boot connection;
- minimal example;
- production example;
- failure examples;
- troubleshooting commands;
- anti-patterns;
- CKAD shortcuts;
- production checklist;
- official sources.

