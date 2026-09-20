# Walkthrough — REST + PostgreSQL + HPA + NetworkPolicy

## Архитектура

```text
caller
  ↓
Service orders-api
  ↓
Spring Boot Pods
  ├── Hikari -> postgres-rw:5432
  ├── HPA <- CPU metrics
  └── NetworkPolicy -> allowed egress
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [Spring config](application.yaml)
- [HPA reference](../../docs/knowledge/manifests/hpa.md)
- [NetworkPolicy reference](../../docs/knowledge/manifests/networkpolicy.md)

## CPU request -> HPA

```text
request = 500m
usage   = 350m
utilization ≈ 70%
```

CPU request одновременно влияет на:
- scheduling;
- autoscaling denominator.

## Hikari -> DB capacity

```text
maxReplicas=20
pool=10
≈ up to 200 client connections
```

### Осторожно

HPA может успешно спасти HTTP CPU и одновременно перегрузить PostgreSQL.

## NetworkPolicy

Default deny egress изолирует Pods.

После него нужно явно разрешить:
- DNS;
- PostgreSQL;
- payment API;
- telemetry/OIDC where needed.

В `all.yaml` DNS rule намеренно не hardcoded, потому что cluster DNS labels vary.

## Liveness vs DB

Плохая модель:

```text
DB down
 -> liveness down
 -> all Pods restart
 -> reconnect storm
```

DB dependency обычно не должна быть liveness condition.

## Failure scenarios

- CPU requests missing -> HPA metric trouble;
- DB password wrong -> auth failure;
- PostgreSQL egress denied -> TCP failure;
- DNS denied -> UnknownHost;
- pool exhausted -> request latency.

## Проверка

```bash
kubectl get hpa
kubectl describe hpa orders-api
kubectl top pod
kubectl get networkpolicy
kubectl exec <pod> -- nslookup postgres-rw
```
