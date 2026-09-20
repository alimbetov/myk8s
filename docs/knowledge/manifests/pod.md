# Manifest Reference — Pod

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Purpose
Pod — минимальная schedulable workload unit Kubernetes.

## apiVersion / kind
- apiVersion: v1
- kind: Pod

## metadata
- name
- namespace
- labels
- annotations

## spec core
- containers
- initContainers
- restartPolicy
- serviceAccountName
- automountServiceAccountToken
- terminationGracePeriodSeconds
- securityContext
- volumes
- nodeName
- nodeSelector
- affinity
- tolerations
- topologySpreadConstraints
- imagePullSecrets
- dnsPolicy
- hostNetwork
- hostPID
- hostIPC

## container core
- name
- image
- imagePullPolicy
- command
- args
- workingDir
- ports
- env/envFrom
- resources
- probes
- lifecycle
- securityContext
- volumeMounts

## Status concepts
- phase
- conditions
- containerStatuses
- restartCount
- waiting/running/terminated state
- Ready

## Defaults to document fully
- restartPolicy
- terminationGracePeriodSeconds
- dnsPolicy
- imagePullPolicy rules

## Spring Boot links
- JVM process
- local filesystem ephemeral
- probes
- SIGTERM
- stdout/stderr

## Troubleshooting
```bash
kubectl get pod -o wide
kubectl describe pod <name>
kubectl logs <name>
kubectl logs <name> --previous
kubectl exec -it <name> -- sh
```

## Sources
- https://kubernetes.io/docs/concepts/workloads/pods/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
