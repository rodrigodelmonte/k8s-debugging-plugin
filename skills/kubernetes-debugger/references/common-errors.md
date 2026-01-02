# Common Kubernetes Errors

## Scheduling Errors

### "0/N nodes are available"
**Causes:**
- `Insufficient cpu`: Nodes lack CPU capacity
- `Insufficient memory`: Nodes lack memory capacity
- `node(s) had taint that pod didn't tolerate`: Taint/toleration mismatch
- `node(s) didn't match node selector`: Label mismatch
- `node(s) didn't match pod affinity/anti-affinity`: Affinity rules
- `node(s) had volume node affinity conflict`: PV zone mismatch

**Solutions:**
```
# Check node capacity
kubectl_get(resourceType="nodes", output="wide")
kubectl_describe(resourceType="node", name="<node>")

# Check pod requirements
kubectl_describe(resourceType="pod", name="<pending-pod>")

# Look at requests vs allocatable in node describe
```

### "persistentvolumeclaim not found" / "unbound"
**Solutions:**
```
kubectl_get(resourceType="pvc", namespace="<ns>")
kubectl_describe(resourceType="pvc", name="<pvc>")
# Check storage class exists and has provisioner
kubectl_get(resourceType="storageclass")
```

## Image Errors

### "ImagePullBackOff" / "ErrImagePull"
**Common messages:**
- `repository does not exist`: Wrong image name
- `manifest unknown`: Tag doesn't exist
- `unauthorized`: Auth required, missing imagePullSecrets
- `connection refused`: Registry unreachable

**Debug steps:**
```
kubectl_describe(resourceType="pod") # Check exact image name
# Verify image exists locally:
exec_in_pod(command=["crictl", "pull", "<image>"])  # On node
```

### "rpc error: code = Unknown desc = failed to pull"
Usually network issue reaching registry. Check:
- Node firewall rules
- Proxy settings
- Registry DNS resolution

## Container Runtime Errors

### "CrashLoopBackOff"
**Debug:**
```
kubectl_logs(previous=true)  # Crashed container logs
kubectl_describe  # Exit code and reason
```

**By exit code:**
- 1: Application error - check logs
- 137: OOMKilled - increase memory limit
- 143: SIGTERM - check preStop hooks, graceful shutdown

### "CreateContainerConfigError"
**Causes:**
- ConfigMap not found
- Secret not found
- Key missing in ConfigMap/Secret

**Debug:**
```
kubectl_describe(resourceType="pod")  # Shows which ConfigMap/Secret missing
kubectl_get(resourceType="configmaps", namespace="<ns>")
kubectl_get(resourceType="secrets", namespace="<ns>")
```

### "OOMKilled"
Container exceeded memory limit.

**Solutions:**
1. Increase `resources.limits.memory`
2. Fix memory leak in application
3. Set appropriate `resources.requests.memory` for scheduling

## Network Errors

### "connection refused" to Service
**Debug:**
```
kubectl_get(resourceType="endpoints", name="<svc>")  # Should have IPs
kubectl_describe(resourceType="service", name="<svc>")  # Check selector
kubectl_get(resourceType="pods", labelSelector="<selector>")  # Matching pods
```

### "no such host" / DNS failures
**Debug:**
```
# Check CoreDNS
kubectl_get(resourceType="pods", namespace="kube-system", labelSelector="k8s-app=kube-dns")

# Test from pod
exec_in_pod(command=["nslookup", "kubernetes.default"])
exec_in_pod(command=["cat", "/etc/resolv.conf"])
```

### "context deadline exceeded"
Network timeout. Check:
- NetworkPolicies blocking traffic
- Service port matching target port
- Target pods are healthy

## Probe Failures

### "Liveness probe failed"
Container will be killed and restarted.

**Debug:**
```
kubectl_describe  # Shows probe config and failure message
kubectl_logs  # Application state when probe failed
```

### "Readiness probe failed"
Pod removed from Service endpoints.

**Common causes:**
- App not ready yet (increase `initialDelaySeconds`)
- App overloaded
- Wrong probe path/port

## RBAC Errors

### "forbidden: User cannot"
ServiceAccount lacks permissions.

**Debug:**
```
kubectl_get(resourceType="serviceaccount", name="<sa>", namespace="<ns>")
kubectl_describe(resourceType="clusterrolebinding")
kubectl_describe(resourceType="rolebinding", namespace="<ns>")
```

## Resource Quota Errors

### "exceeded quota"
Namespace quota exhausted.

**Debug:**
```
kubectl_describe(resourceType="resourcequota", namespace="<ns>")
```
