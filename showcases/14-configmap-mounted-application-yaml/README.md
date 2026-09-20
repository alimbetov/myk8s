# Showcase 14 — ConfigMap mounted as application.yaml

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
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
ConfigMap key application.yaml
        |
        v
volume
        |
        v
/etc/spring/application.yaml
        |
        v
SPRING_CONFIG_ADDITIONAL_LOCATION=file:/etc/spring/
        |
        v
Spring Boot Environment
```

## Почему использовать file instead of env

Удобно, когда configuration:
- hierarchical;
- большая;
- хорошо читается YAML;
- содержит списки/maps;
- должна выглядеть почти так же, как локальный application.yaml.

## ConfigMap

```yaml
data:
  application.yaml: |
    app:
      customer-api:
        base-url: http://customer-api:8080
        connect-timeout: 1s
```

Каждый ConfigMap key при volume mount становится file.

## Volume mapping

```text
ConfigMap.metadata.name
      ↓
volumes[].configMap.name

volume.name
      =
volumeMount.name

mountPath
      ↓
/etc/spring/application.yaml
```

## Spring loading

Container получает:

```yaml
SPRING_CONFIG_ADDITIONAL_LOCATION: file:/etc/spring/
```

Spring Boot добавляет external directory к config search locations.

## Update semantics

Kubernetes eventually обновляет обычный mounted ConfigMap volume после изменения ConfigMap.

Но:

> Spring ApplicationContext не обязан автоматически перечитать application.yaml.

Поэтому operational patterns:
- immutable/versioned ConfigMap + rollout;
- explicit refresh mechanism, если он действительно нужен.

## subPath trap

Если mount сделать через `subPath`, ConfigMap automatic projection updates не приходят.

## Failure simulations

1. ConfigMap missing -> Pod не стартует.
2. YAML syntax invalid -> Spring startup fails.
3. Property type invalid -> binding/validation startup failure.
4. ConfigMap changed, Pod not restarted -> file может обновиться, но Spring still uses old in-memory config.
5. `subPath` used expecting live updates -> file remains old.

## Sources

- Kubernetes ConfigMap volume semantics.
- Spring Boot external configuration/additional locations.
