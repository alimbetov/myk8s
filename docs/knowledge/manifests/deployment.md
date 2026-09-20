# Manifest Reference — Deployment

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Purpose
Deployment управляет ReplicaSet и поддерживает desired state для stateless Pods.

## apiVersion / kind
- apiVersion: apps/v1
- kind: Deployment

## metadata
- name
- namespace
- labels
- annotations

## spec core
- replicas
- selector.matchLabels
- template.metadata.labels
- template.spec
- strategy.type
- strategy.rollingUpdate.maxUnavailable
- strategy.rollingUpdate.maxSurge
- minReadySeconds
- progressDeadlineSeconds
- revisionHistoryLimit
- paused

## template.spec core
- serviceAccountName
- automountServiceAccountToken
- terminationGracePeriodSeconds
- securityContext
- containers
- initContainers
- volumes
- nodeSelector
- affinity
- tolerations
- topologySpreadConstraints

## container core
- name
- image
- imagePullPolicy
- command
- args
- ports
- env / envFrom
- resources
- startupProbe
- livenessProbe
- readinessProbe
- lifecycle
- securityContext
- volumeMounts

## Defaults to document fully in iteration 2
- replicas
- strategy.type
- maxUnavailable
- maxSurge
- progressDeadlineSeconds
- revisionHistoryLimit
- terminationGracePeriodSeconds
- imagePullPolicy rules

## Runtime relationships
Deployment -> ReplicaSet -> Pod -> Container.

## Spring Boot links
- image/JVM
- env/config
- probes
- graceful shutdown
- resources
- rollout/schema compatibility

## Troubleshooting commands
```bash
kubectl get deploy,rs,pod
kubectl describe deploy <name>
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>
kubectl get events --sort-by=.lastTimestamp
```

## CKAD
Must know selector/template labels, replicas, strategy, image update, probes/resources.

## Sources
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
