# Showcase Documentation Standard

Каждый production-like стенд в `showcases/` должен состоять минимум из:

- `README.md` — архитектура, цель и навигация;
- `all.yaml` — связный Kubernetes пример;
- `application.yaml` — Spring Boot contract, если он нужен;
- `WALKTHROUGH.md` — подробный педагогический разбор.

## README должен отвечать

1. Что строим?
2. Как выглядело бы на VM/до Kubernetes?
3. Какие Kubernetes objects участвуют?
4. Какой request/data flow?
5. Какие файлы открыть?
6. Какие production assumptions есть?
7. Что намеренно сломать?

## WALKTHROUGH должен содержать

### 1. Схему

Не только список ресурсов, а flow:

```text
caller
 -> DNS
 -> Service
 -> EndpointSlice
 -> Ready Pod
 -> Spring Boot
 -> PostgreSQL
```

### 2. Mapping table

Например:

| Откуда | Куда | Тип связи |
|---|---|---|
| Service.selector | Pod.labels | label selector |
| Service.targetPort | containerPort.name | named port |
| HPA.scaleTargetRef | Deployment.name | exact object ref |
| ConfigMap key | Spring property | runtime configuration |

### 3. Annotated example

Каждый важный fragment YAML сопровождается:
- зачем он нужен;
- что произойдёт runtime;
- какой default;
- где ошибаются;
- как проверить.

### 4. Runtime sequence

Например:

```text
kubectl apply
 -> API Server stores objects
 -> Deployment controller creates ReplicaSet
 -> scheduler chooses node
 -> kubelet starts container
 -> startup/readiness pass
 -> EndpointSlice marks endpoint ready
 -> Service can route
```

### 5. Production attention blocks

Обязательные пометки:
- **Важно**
- **Осторожно**
- **Production**
- **Не путать**
- **Проверка**

### 6. Failure simulations

Минимум 3:
- configuration failure;
- networking/routing failure;
- runtime/dependency failure.

### 7. Links

README должен содержать относительные ссылки на все файлы стенда.

## Правило качества

Если пример нельзя объяснить словами “что произойдёт после apply и почему”, пример ещё не готов.
