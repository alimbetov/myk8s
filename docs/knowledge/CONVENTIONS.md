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


## Textbook-first rule

Каждая новая knowledge-глава должна начинаться с педагогического слоя до глубокого technical reference.

Обязательный порядок:

```text
1. Место темы в общей Kubernetes-системе
2. Схема объектов и связей
3. Runtime sequence
4. Annotated manifest / Spring fragment
5. Только затем подробная теория и field-by-field details
6. Ссылка на production-like showcase
7. Failure scenarios / troubleshooting
```

### Context diagram

Глава должна отвечать:

> Где этот механизм находится относительно Deployment, Pod, Service, Spring Boot и dependencies?

Пример:

```text
Internet
   ↓
Gateway
   ↓
Service
   ↓
Pod
   ↓
Spring Boot
   ↓
PostgreSQL
```

Из схемы выделяется именно та часть, которой посвящена глава.

### Object relationship diagram

Показывает конкретные Kubernetes mappings:

```text
Service.selector
      =
Pod.labels

Service.targetPort
      =
containerPort.name
```

### Runtime diagram

Обязателен для lifecycle mechanisms:

```text
readiness success
   ↓
Pod Ready=True
   ↓
EndpointSlice ready=true
   ↓
Service sends traffic
```

### Annotated manifests

У каждого крупного showcase должны существовать два варианта:

- `annotated.yaml` — учебный YAML с объясняющими комментариями;
- `all.yaml` — чистый apply-ready reference.

Комментарии в `annotated.yaml` должны объяснять:
- зачем поле;
- с чем связано;
- runtime effect;
- production caveat;
- типичную ошибку.

Не превращать `all.yaml` в длинный учебник: чистый вариант нужен для практики.

### Attention markers

В prose и comments использовать явно:

- **ВАЖНО** — ключевой invariant;
- **ОСТОРОЖНО** — настройка с опасным побочным эффектом;
- **PRODUCTION** — что обязательно переосмыслить вне lab;
- **НЕ ПУТАТЬ** — похожие, но разные concepts;
- **ПРОВЕРКА** — kubectl/application verification.

### Quality test

Материал не считается педагогически готовым, если читатель после раздела не может ответить:

1. Где этот объект расположен в общей архитектуре?
2. С какими ресурсами он связан?
3. Какими полями происходит mapping?
4. Что происходит после `kubectl apply`?
5. Что сломается при неправильной настройке?
6. Как это проявится в Spring Boot?
7. Как это проверить командами?
