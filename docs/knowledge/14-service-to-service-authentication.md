# 14 — Service-to-service authentication

## Учебная карта темы

### Где authentication находится в internal call

```text
orders-api
   |
   | 1 DNS / Service
   | 2 NetworkPolicy
   | 3 TLS/mTLS
   | 4 OAuth2/JWT
   v
payment-api
   |
   v
Authorization
```

Каждый слой отвечает на другой вопрос:

```text
NetworkPolicy -> может ли packet дойти?
TLS           -> защищён ли transport?
mTLS          -> какая workload identity?
JWT           -> какой application/user principal?
Authorization -> разрешено ли действие?
```

### Spring Resource Server fragment

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OIDC_ISSUER_URI}
```

### Ошибки по слоям

```text
UnknownHost -> DNS
timeout     -> network
TLS failure -> transport trust
401         -> authentication
403         -> authorization
```

Практика: [Showcase 06](../../showcases/06-service-to-service-security/README.md).

Проверено: 2026-09-20.

Network reachability и identity — разные задачи. Если Pod может открыть TCP connection к другому Pod, это ещё не означает, что ему можно доверять.

## 1. Security layers

```text
NetworkPolicy -> может ли соединиться
TLS          -> зашифрован ли канал
mTLS         -> кто workload peer
JWT/OAuth2   -> кто principal/user/service
Authorization-> что ему разрешено
```

## 2. Почему internal network не trusted

Старая модель:

```text
inside corporate network = trusted
```

Проблема: compromised internal workload получает lateral movement.

Современная модель:
- authenticate;
- authorize;
- least privilege;
- network isolation как дополнительный слой.

## 3. OAuth2 Resource Server

Spring Boot/Spring Security:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OIDC_ISSUER_URI}
```

API валидирует token signature/issuer/claims according to configured provider.

## 4. Service identity vs user identity

Request может идти:

```text
user
 -> frontend
 -> orders-service
 -> customer-service
```

customer-service может захотеть знать:
- какой workload вызвал;
- какой end user инициировал;
- какие scopes/roles.

Это две разные identity dimensions.

## 5. Token propagation

Blind propagation user token через все services часто создаёт overly broad trust.

Нужно решить:
- delegation;
- token exchange;
- service account/client credentials;
- audience;
- scope minimization.

## 6. Client credentials

Machine-to-machine flow:

```text
service A
 -> identity provider
 -> access token for API B
 -> API B validates
```

Credentials client приложения должны быть Secret/external secret manager.

## 7. mTLS

mTLS:
- encrypts;
- authenticates both TLS peers;
- can provide workload identity.

Но certificate identity не автоматически означает business role.

```text
mTLS says: caller = orders workload
authorization says: orders may call /internal/reserve
```

## 8. Service mesh

Mesh может автоматизировать:
- mTLS;
- certificate rotation;
- traffic policy;
- telemetry.

Но добавляет:
- control plane;
- proxies/ambient data plane;
- operational complexity;
- another debugging layer.

Не вводите mesh только ради модного mTLS, если requirements проще.

## 9. JWT audience

Token должен быть предназначен для нужной API audience. Это снижает риск reuse token не в том сервисе.

## 10. Authorization

Пример Spring:

```java
@PreAuthorize("hasAuthority('SCOPE_customer.read')")
public Customer getCustomer(...) { ... }
```

Но business authorization может быть глубже:
- tenant;
- ownership;
- account relation.

## 11. Kubernetes ServiceAccount token

Не путать с application JWT.

Kubernetes ServiceAccount token предназначен прежде всего для Kubernetes/workload identity scenarios. Не следует автоматически использовать его как business auth token между сервисами без выбранной architecture.

## 12. Secret rotation

OAuth client secret/certificate rotation должен быть runbook:
1. issue new;
2. deploy;
3. verify;
4. revoke old.

## 13. Failure practice: expired token

Expected:
- network works;
- TLS works;
- HTTP 401.

Это authentication failure, не NetworkPolicy.

## 14. Failure practice: insufficient scope

Expected:
- authenticated;
- HTTP 403.

Это authorization failure.

## 15. Failure practice: expired certificate

Expected:
- TLS handshake fails до HTTP layer.

Диагностика должна разделять transport/auth.

## 16. Logging

Логировать можно:
- subject/client id;
- scopes;
- token issuer;
- correlation id.

Не логировать:
- raw bearer token;
- private key;
- client secret.

## 17. Anti-patterns

- «внутри cluster auth не нужен»;
- API key один на все services;
- eternal non-rotated secrets;
- JWT without audience/issuer validation;
- NetworkPolicy used instead of authorization;
- logging Authorization header.

## 18. Developer vs Platform

| Область | Developer | Platform/Security | Shared |
|---|---:|---:|---:|
| Spring auth config | ✓ | | ✓ |
| IdP | | ✓ | |
| authorization | ✓ | | ✓ |
| mTLS platform | | ✓ | |
| token scopes | | | ✓ |
| rotation | | | ✓ |

## 19. CKAD

Application authentication почти выходит за CKAD. Экзамен ближе к ServiceAccount/Secret/NetworkPolicy. Production требует OAuth2/mTLS/identity architecture.

## Связанные production-like примеры

- [NetworkPolicy + OAuth2/JWT layers](../../showcases/06-service-to-service-security/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/
- https://kubernetes.io/docs/concepts/security/service-accounts/

Проверено: **2026-09-20**.
