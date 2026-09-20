# Showcase 15 — Kubernetes Secret mounted as Spring Boot configtree

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Место этого стенда в общей системе

```text
Secret
  |
  v
Pod-mounted files
  |
  v
Spring configtree
  |
  v
Spring Environment
  |
  v
DB/client credentials
```

Configtree — это способ доставки Secret в Spring, а не отдельная система secret management.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

Проверено: 2026-09-20.

## Схема

```text
Secret keys
 spring.datasource.username
 spring.datasource.password
          |
          v
mounted files
/etc/secrets/
          |
          v
spring.config.import=configtree:/etc/secrets/
          |
          v
Spring Environment
```

## Почему configtree

Вместо environment variables каждый Secret key становится отдельным file.

Spring Boot умеет импортировать directory через:

```yaml
spring:
  config:
    import: configtree:/etc/secrets/
```

или environment:

```text
SPRING_CONFIG_IMPORT=configtree:/etc/secrets/
```

## Dot notation

Secret key:

```text
spring.datasource.password
```

становится Spring property с тем же именем.

## Manifest mapping

```text
Secret.metadata.name
      ↓
volume.secret.secretName

volume.name
      =
volumeMount.name

files
      ↓
Spring configtree
```

## Почему это удобно

- secrets не перечисляются в process environment;
- хорошо для external secret/CSI integrations;
- естественно для TLS/private keys;
- property names могут быть прямыми Spring keys.

## Rotation

Kubernetes mounted Secret eventually updates files, но Spring client/pool не обязательно reload credentials.

Rotation всё равно требует explicit application behavior или rollout.

## Failure simulations

1. Secret missing -> Pod startup blocked.
2. Required file missing -> Spring property unresolved/binding failure.
3. Secret changed -> file changes, Hikari still uses old credentials until reload/new pool behavior.
4. Wrong mount path -> configtree directory absent.
5. `optional:configtree:` accidentally used for required secrets -> application may start with unintended fallback.

## Sources

- Spring Boot Externalized Configuration — Configuration Trees.
- Kubernetes Secret volume behavior.
