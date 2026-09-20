# Documentation conventions

Проверено: 2026-09-20.

Каждая knowledge-глава проекта `myk8s` должна быть **самодостаточной**: основной материал должен быть понятен без обязательного перехода по внешним ссылкам.

## Обязательные блоки

### Mental model
Что происходит в системе и кто за что отвечает.

### Как было раньше / как сейчас
Связь привычной VM/монолитной модели с Kubernetes.

### Почему так
Причина существования механизма, а не только синтаксис YAML.

### Практический пример
Конкретный Spring Boot configuration, Kubernetes manifest или sequence команд.

### Что произойдёт
Пошаговая runtime sequence: controller → kubelet → Pod → Service → application.

### Failure practice
Намеренная поломка, наблюдаемый symptom, диагностика и исправление.

### Anti-pattern
Что выглядит рабочим, но плохо масштабируется или создаёт operational/security risk.

### Production checklist
Что проверить перед реальным использованием.

### CKAD mapping
Что нужно уметь сделать быстро на экзамене и что остаётся production-only knowledge.

### Sources
Источники нужны для проверки спецификации и углубления, **не вместо объяснений**.

## Формат примеров

Предпочтительная цепочка:

```text
problem
 -> manifest/application configuration
 -> expected runtime behavior
 -> how to verify
 -> how to break
 -> how to diagnose
 -> how to repair
```

## Правило для значений по умолчанию

Если Kubernetes/Spring имеет значимый default, документ обязан указать:
- default;
- когда он удобен;
- когда его нужно переопределить;
- какой operational effect будет при изменении.

## Правило security

Всегда разделять:
- network reachability;
- transport security;
- authentication;
- authorization;
- Kubernetes API RBAC;
- secret lifecycle.

## Правило stateful

Stateful chapter обязан объяснять:
- где реально лежат data;
- что переживает Pod recreation;
- что переживает node loss;
- кто отвечает за replication/quorum;
- backup и restore;
- RPO/RTO;
- upgrade/failover path.

## Правило терминологии

В тексте разрешено оставлять Kubernetes/Java термины на английском, когда перевод делает понятие менее точным: Pod, Deployment, rollout, readiness, liveness, Secret, Service, operator, failover, pool, timeout.
