# Manifest Reference — ConfigMap

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Purpose
Несекретная runtime configuration.

## apiVersion / kind
- apiVersion: v1
- kind: ConfigMap

## metadata
- name
- namespace
- labels
- annotations

## data forms
- data
- binaryData
- immutable

## Consumption
- env.valueFrom.configMapKeyRef
- envFrom.configMapRef
- volumes.configMap
- projected volume

## Runtime behavior to detail
- env values fixed at process start
- mounted file update behavior
- optional refs
- missing key/object behavior
- immutable ConfigMap

## Spring Boot links
- externalized configuration
- relaxed binding
- @ConfigurationProperties
- application.yaml placeholders

## Troubleshooting
```bash
kubectl get configmap
kubectl describe configmap <name>
kubectl get configmap <name> -o yaml
kubectl describe pod <pod>
```

## Sources
- https://kubernetes.io/docs/concepts/configuration/configmap/
