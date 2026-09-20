# 02 — Services, DNS and service-to-service security

Проверено: 2026-09-20.

## Mental model

Deployment создаёт меняющиеся Pods. Service даёт стабильную точку обнаружения.

```text
orders Pod
  -> http://customer-api:8080
  -> cluster DNS
  -> Service customer-api
  -> EndpointSlice
  -> Ready customer-api Pods
```

В одном namespace обычно достаточно `http://customer-api:8080`. Между namespaces: `customer-api.crm.svc.cluster.local`.

Не используйте Pod IP в application config.

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
spec:
  selector:
    app: orders
  ports:
    - name: http
      port: 8080
      targetPort: http
```

Service selector должен совпадать с Pod labels. Проверка:

```bash
kubectl get svc orders
kubectl get endpointslice -l kubernetes.io/service-name=orders
kubectl exec deploy/debug -- nslookup orders
kubectl exec deploy/debug -- curl -v http://orders:8080/readyz
```

## NetworkPolicy

Default deny:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

После default deny разрешения должны быть явными, включая DNS egress.

NetworkPolicy работает только если CNI plugin её реально поддерживает.

## Authentication is separate

NetworkPolicy отвечает: «может ли packet пройти?»

OAuth2/JWT/mTLS/application ACL отвечает: «кто caller и что ему разрешено?»

Для банковского/enterprise microservice backend типичный stack:

```text
NetworkPolicy
+ TLS
+ OAuth2 Resource Server / workload identity
+ authorization by scopes/roles/claims
+ audit
```

mTLS полезен для workload identity, но сам по себе не описывает business authorization пользователя.

## Spring Boot example

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OIDC_ISSUER_URI}
```

Service DNS URL:

```yaml
clients:
  customer:
    base-url: http://customer-api:8080
```

Если platform предоставляет service mesh/mTLS, приложение всё равно должно валидировать user/service authorization на нужном уровне.

## Failure scenarios

**DNS failure**
```bash
kubectl exec <pod> -- getent hosts customer-api
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl get svc customer-api
kubectl get endpointslice -l kubernetes.io/service-name=customer-api
```

**Network denied**
- DNS resolves;
- TCP connect times out/denied;
- inspect policies in source and destination namespaces.

**No endpoints**
- Service существует;
- DNS работает;
- selector не совпадает или все Pods unready.

## Sources

- https://kubernetes.io/docs/concepts/services-networking/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/concepts/security/service-accounts/
- https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/index.html

Проверено: **2026-09-20**.
