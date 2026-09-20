# Showcase 16 — initContainer + main Spring Boot container

Проверено: 2026-09-20.

## Схема

\`\`\`text
Pod starts
   |
   v
initContainer prepare-config
   |
   | reads ConfigMap template
   | writes generated file
   v
shared emptyDir
   |
   v
main Spring Boot container
   |
   | loads /work/config/application.yaml
   v
Ready
\`\`\`

## Зачем initContainer

Подходит для one-time preparation перед запуском application:

- преобразовать template;
- подготовить filesystem;
- загрузить/сгенерировать локальный runtime artifact;
- выполнить lightweight bootstrap;
- проверить prerequisite, если failure semantics осознанны.

Init container **обязан завершиться успешно** до запуска main container.

## Mapping

ConfigMap:

\`\`\`text
application.template.yaml
\`\`\`

mount:

\`\`\`text
/templates/application.template.yaml
\`\`\`

initContainer записывает:

\`\`\`text
/work/config/application.yaml
\`\`\`

Main container монтирует тот же \`emptyDir\` и читает:

\`\`\`text
SPRING_CONFIG_ADDITIONAL_LOCATION=file:/work/config/
\`\`\`

## Shared volume

\`\`\`text
volumes[].name=runtime-config
      =
initContainer.volumeMounts[].name
      =
main.volumeMounts[].name
\`\`\`

Так containers обмениваются файлами без внешнего storage.

## Почему emptyDir

Generated file нужен только на время жизни Pod.

При replacement Pod initContainer создаст его заново.

Это хороший пример ephemeral derived state.

## Не использовать initContainer как вечный wait loop

Плохо:

\`\`\`sh
while ! nc db 5432; do sleep 5; done
\`\`\`

если это скрывает dependency architecture.

Для обычных external dependencies:
- finite connection timeout;
- retry/backoff;
- readiness;
- startupProbe;
обычно лучше.

## Не делать DB migration без coordination

Если Deployment replicas=5, initContainer migration может стартовать одновременно в 5 Pods.

Liquibase/Flyway locking может помочь, но ownership migration должен быть спроектирован отдельно.

## Failure simulations

1. ConfigMap missing -> init container не стартует.
2. init script exits non-zero -> main container никогда не запустится.
3. generated YAML invalid -> init succeeds, Spring fails.
4. wrong volume name -> file не попадает в main.
5. Pod deleted -> emptyDir исчезает, replacement генерирует файл снова.

## Диагностика

\`\`\`bash
kubectl get pod
kubectl describe pod <pod>
kubectl logs <pod> -c prepare-config
kubectl logs <pod> -c app
\`\`\`

## Sources

- https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
