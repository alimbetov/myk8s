# Showcase 01 — Internal Spring Boot REST service

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Полный связный manifest](all.yaml)
- [Spring Boot configuration](application.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Цель

Базовая production-like схема внутреннего Spring Boot API, который доступен только внутри cluster.

```text
caller Pod
   |
   | http://customer-api:8080
   v
Service customer-api
   |
   | selector app.kubernetes.io/name=customer-api
   v
EndpointSlice
   |
   +--> customer-api Pod #1 Ready
   +--> customer-api Pod #2 Ready
```

## Objects

- ConfigMap
- Secret
- ServiceAccount
- Deployment
- Service

## Главные mapping'и

### 1. Deployment labels -> Service selector

Deployment:

```yaml
template:
  metadata:
    labels:
      app.kubernetes.io/name: customer-api
```

Service:

```yaml
selector:
  app.kubernetes.io/name: customer-api
```

Если значения расходятся, Service существует, DNS работает, но ready backend отсутствует.

### 2. container port -> Service targetPort

Deployment:

```yaml
ports:
  - name: http
    containerPort: 8080
```

Service:

```yaml
ports:
  - port: 8080
    targetPort: http
```

Caller обращается к Service port `8080`; Service направляет traffic на named port `http`, который в Pod равен `8080`.

### 3. ConfigMap -> Spring Boot

```yaml
data:
  CUSTOMER_CACHE_TTL: 60s
```

Deployment использует `envFrom`, а `application.yaml` читает:

```yaml
app:
  customer-cache-ttl: ${CUSTOMER_CACHE_TTL:30s}
```

### 4. Secret -> datasource password

Secret key:

```text
DB_PASSWORD
```

становится environment variable и затем Spring property.

### 5. readiness -> Service endpoints

```text
/readyz fails
 -> Pod Ready=False
 -> EndpointSlice ready=false
 -> Service прекращает обычный traffic в этот Pod
```

## Проверка

```bash
kubectl apply -f all.yaml

kubectl get deploy,pod,svc
kubectl get endpointslice   -l kubernetes.io/service-name=customer-api

kubectl run curl --rm -it   --image=curlimages/curl --   curl -v http://customer-api:8080/actuator/health
```

## Что намеренно сломать

1. Поменять Service selector на `customer-apix`.
2. Поменять targetPort на `9999`.
3. Поменять readiness path на `/broken`.
4. Удалить Secret и выполнить rollout restart.

Каждый failure ломает **разный слой**, хотя пользователь видит одно и то же: API недоступно.
