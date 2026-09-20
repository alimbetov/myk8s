# Showcase 13 — Spring Boot + S3-compatible object storage

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Spring Boot configuration](application.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

Проверено: 2026-09-20.

## Схема

```text
Spring Boot
   |
   | S3 API
   v
object-storage endpoint
   |
   +--> bucket uploads
   +--> bucket reports
```

## Когда object storage лучше PVC

Для:
- uploads;
- generated reports;
- attachments;
- exports;
- assets;

object storage обычно удобнее shared filesystem, потому что application работает с object key, а не с конкретным mounted disk.

## Configuration split

ConfigMap:

```text
S3_ENDPOINT
S3_REGION
S3_BUCKET
```

Secret:

```text
S3_ACCESS_KEY
S3_SECRET_KEY
```

Это разделяет routing/non-sensitive config и credentials.

## Spring mapping

```yaml
app:
  storage:
    endpoint: ${S3_ENDPOINT}
    region: ${S3_REGION}
    bucket: ${S3_BUCKET}
    access-key: ${S3_ACCESS_KEY}
    secret-key: ${S3_SECRET_KEY}
```

## Endpoint variants

Managed cloud:

```text
https://s3.<region>....
```

In-cluster S3-compatible storage:

```text
http://object-storage:9000
```

Для MinIO/RustFS deployment/operator нужен отдельный stateful design.

## Key design

Не хранить filename как physical path assumption.

Пример:

```text
tenant/42/uploads/2026/09/<uuid>.pdf
```

Metadata:
- object key;
- original filename;
- content type;
- size;
- checksum;
- createdAt;
- expiresAt;
- status.

обычно хранится в PostgreSQL.

## Presigned URLs

Для больших файлов:

```text
browser
 -> Spring API asks for permission
 -> application returns presigned URL
 -> browser uploads directly to object storage
```

Это уменьшает network/memory pressure на Spring Pods.

## Failure simulations

1. Wrong endpoint -> DNS/connect error.
2. Wrong credentials -> 401/403-like S3 auth failure.
3. Bucket missing -> application-level storage error.
4. Object exists but DB metadata transaction failed -> consistency problem.
5. DB row exists but upload failed -> orphan/incomplete state.

## Consistency

PostgreSQL transaction и S3 object write не являются одной ACID transaction.

Нужны patterns:
- status=PENDING -> upload -> READY;
- cleanup orphan objects;
- idempotent object keys;
- retry policy.

## NetworkPolicy

Для external S3 обычная Kubernetes NetworkPolicy может быть неудобна для FQDN-based egress. CNI-specific FQDN policy либо egress gateway может быть platform concern.
