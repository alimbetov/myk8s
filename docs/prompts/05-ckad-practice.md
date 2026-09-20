# Prompt 05 — CKAD Practice Track

Цель — параллельно с production-обучением системно готовиться к Certified Kubernetes Application Developer (CKAD).

## Правила

Перед каждым обновлением проверить актуальный официальный curriculum CNCF/Linux Foundation и текущую exam environment information.

Не смешивать:
- знания для production;
- навыки, реально проверяемые на CKAD.

## Структура подготовки

Для каждой exam domain:

1. concepts;
2. imperative kubectl commands;
3. YAML patterns;
4. time-saving techniques;
5. debugging;
6. timed labs;
7. failure injection;
8. final checklist.

Текущие официальные домены должны проверяться перед генерацией плана. На момент создания framework ориентир:
- Application Design and Build;
- Application Deployment;
- Application Observability and Maintenance;
- Application Environment, Configuration and Security;
- Services and Networking.

## Практический формат

Каждая лаборатория:
- ограничение времени;
- задача без готового manifest;
- проверочная команда;
- expected state;
- hidden trap;
- cleanup.

## Обязательные навыки

Проверить против текущего curriculum и покрыть как минимум:
- Pods;
- Deployments;
- Jobs/CronJobs;
- ConfigMaps;
- Secrets;
- ServiceAccounts/security context;
- probes;
- resources;
- Services;
- NetworkPolicy;
- volumes;
- rollout;
- logs;
- exec;
- describe/events;
- troubleshooting.

## Экзаменационный журнал

Создать в дальнейшем:
`docs/ckad/progress.md`

Таблица:
`skill | confidence 0-3 | last lab | elapsed time | mistake | repeat date`.

## Deliverables
- `docs/ckad/roadmap.md`;
- `docs/ckad/progress.md`;
- `labs/ckad/`.
