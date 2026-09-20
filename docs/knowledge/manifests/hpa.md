# Manifest Reference — HorizontalPodAutoscaler

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## apiVersion / kind
- apiVersion: autoscaling/v2
- kind: HorizontalPodAutoscaler

## spec core
- scaleTargetRef
- minReplicas
- maxReplicas
- metrics
- behavior

## metric types
- Resource
- Pods
- Object
- External
- ContainerResource

## target types
- Utilization
- AverageValue
- Value

## behavior core
- scaleUp
- scaleDown
- stabilizationWindowSeconds
- policies
- selectPolicy

## Runtime concepts
- CPU utilization relative to requests
- metrics pipeline
- startup delay
- scale-up/down stabilization
- downstream capacity
- maxReplicas ceiling

## Troubleshooting
```bash
kubectl get hpa
kubectl describe hpa <name>
kubectl top pod
kubectl get deploy
```

## Sources
- https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/
