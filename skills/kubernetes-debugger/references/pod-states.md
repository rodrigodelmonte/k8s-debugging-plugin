# Pod States Reference

## Pod Phases (spec.status.phase)

| Phase | Description |
|-------|-------------|
| Pending | Accepted by cluster, containers not yet ready (scheduling, image pull) |
| Running | Bound to node, all containers created, at least one running |
| Succeeded | All containers terminated successfully, won't restart |
| Failed | All containers terminated, at least one failed |
| Unknown | State cannot be obtained (node communication error) |

## Container States (containerStatuses[].state)

### Waiting
Container not yet running. Check `reason` field:
- `ContainerCreating`: Normal startup
- `CrashLoopBackOff`: Repeated crashes, exponential backoff
- `ImagePullBackOff`: Cannot pull image
- `ErrImagePull`: Image pull failed
- `CreateContainerConfigError`: ConfigMap/Secret missing
- `InvalidImageName`: Malformed image reference

### Running
Container executing. Fields:
- `startedAt`: When container started

### Terminated
Container finished execution. Fields:
- `exitCode`: 0 = success, non-zero = failure
- `reason`: Error, Completed, OOMKilled, etc.
- `startedAt`, `finishedAt`: Timing info

## Common Exit Codes

| Code | Signal | Meaning |
|------|--------|---------|
| 0 | - | Success |
| 1 | - | General application error |
| 126 | - | Command cannot execute |
| 127 | - | Command not found |
| 128+N | N | Terminated by signal N |
| 137 | SIGKILL (9) | OOMKilled or `kubectl delete --force` |
| 143 | SIGTERM (15) | Graceful termination |
| 255 | - | Exit status out of range |

## Pod Conditions

| Condition | True Means |
|-----------|------------|
| PodScheduled | Assigned to a node |
| PodReadyToStartContainers | Sandbox/network ready |
| Initialized | Init containers completed |
| ContainersReady | All containers ready |
| Ready | Pod can serve traffic |

## Debugging by State

### CrashLoopBackOff
```
1. kubectl_logs(previous=true) - get crashed container logs
2. kubectl_describe - check exit code and reason
3. If OOMKilled (137): increase memory limits
4. If exit code 1: application bug, check logs
```

### ImagePullBackOff
```
1. kubectl_describe - check exact image name
2. Verify image exists: check registry
3. Check imagePullSecrets for private registries
4. Check node can reach registry (network/firewall)
```

### Pending
```
1. kubectl_describe - check Events section
2. "Insufficient cpu/memory": scale cluster or reduce requests
3. "didn't match node selector": fix labels
4. "Unschedulable": check taints/tolerations
5. PVC not bound: check storage class/provisioner
```

### ContainerCreating (stuck)
```
1. kubectl_describe - check Events
2. Volume mount issues: PVC not bound, ConfigMap missing
3. Network plugin issues: check CNI pods in kube-system
4. Image pull slow: large image or slow registry
```
