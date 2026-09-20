# Showcase 02 — Public Spring Boot API through Ingress + TLS

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Полный связный manifest](all.yaml)
- [Spring Boot configuration](application.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
Internet
   |
   | HTTPS api.example.kz
   v
Ingress Controller
   |
   | Ingress rule host/path
   v
Service orders-api:8080
   |
   | selector
   v
Ready Pods
```

## Что добавилось относительно Showcase 01

Внутренний `Service` остался `ClusterIP`. Внешний доступ появляется не потому, что Service стал public, а потому что добавился **Ingress + controller + TLS Secret**.

## Mapping

### Ingress backend -> Service

```yaml
backend:
  service:
    name: orders-api
    port:
      name: http
```

должен совпасть с:

```yaml
kind: Service
metadata:
  name: orders-api
spec:
  ports:
    - name: http
```

### TLS host -> certificate

```yaml
tls:
  - hosts:
      - api.example.kz
    secretName: api-example-tls
```

Secret содержит certificate/private key.

### DNS

Public DNS `api.example.kz` должен указывать на внешний endpoint Ingress Controller. Сам Ingress object DNS record не создаёт в стандартном Kubernetes API.

## Platform-specific assumption

В примере:

```yaml
ingressClassName: traefik
```

Это удобно для многих k3s installations, но нужно проверить:

```bash
kubectl get ingressclass
```

Если controller другой — изменить class.

## Проверка

```bash
kubectl get ingress
kubectl describe ingress orders-api
kubectl get svc orders-api
kubectl get endpointslice   -l kubernetes.io/service-name=orders-api
```

## Failure simulations

1. Wrong `ingressClassName`: object есть, controller его не обслуживает.
2. Wrong backend Service name: controller не находит upstream.
3. Service selector mismatch: Ingress и Service есть, endpoints пусты.
4. TLS Secret отсутствует/невалиден: HTTPS конфигурация ломается.
5. Pods NotReady: edge работает, но backend capacity отсутствует.
