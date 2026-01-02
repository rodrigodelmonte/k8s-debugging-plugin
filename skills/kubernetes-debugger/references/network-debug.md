# Network Debugging Reference

## Debug Containers (For Minimal/Distroless Images)

Most production containers don't include debugging tools. Use **ephemeral debug containers** to attach debugging tools to running pods.

### Ephemeral Debug Container (Recommended)
Attach a debug container to an existing pod without restarting it:

```bash
# Using kubectl directly (MCP doesn't have native ephemeral container support)
kubectl debug <pod-name> -it --image=nicolaka/netshoot -n <namespace>

# With process namespace sharing (see processes in other containers)
kubectl debug <pod-name> -it --image=nicolaka/netshoot --share-processes -n <namespace>

# Target a specific container's namespace
kubectl debug <pod-name> -it --image=busybox --target=<container-name> -n <namespace>
```

### Recommended Debug Images

| Image | Size | Tools Included |
|-------|------|----------------|
| `nicolaka/netshoot` | ~300MB | Full networking toolkit (curl, dig, nmap, tcpdump, iperf) |
| `busybox` | ~1MB | Basic utilities (ping, nslookup, wget, nc) |
| `alpine` | ~5MB | apk package manager, basic tools |
| `curlimages/curl` | ~10MB | curl only |

### Debug Sidecar Pod (Alternative)
Deploy a debug pod in the same namespace to test network connectivity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-network
  namespace: <target-namespace>
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
```

Then use `exec_in_pod` on the debug pod:
```
exec_in_pod(name="debug-network", namespace="<ns>", command=["curl", "-v", "http://<service>:<port>"])
```

## Service Debugging Workflow

### 1. Verify Service Configuration
```
kubectl_get(resourceType="service", name="<svc>", namespace="<ns>", output="yaml")
```
Check:
- `spec.selector` matches pod labels
- `spec.ports[].port` (service port)
- `spec.ports[].targetPort` (container port)
- `spec.type` (ClusterIP, NodePort, LoadBalancer)

### 2. Check Endpoints
```
kubectl_get(resourceType="endpoints", name="<svc>", namespace="<ns>")
```
- Empty endpoints = no matching pods or pods not ready
- IPs should match running pod IPs

### 3. Verify Pod Labels
```
kubectl_get(resourceType="pods", namespace="<ns>", labelSelector="app=myapp")
```
Pods must have labels matching service selector AND be Ready.

## DNS Debugging

### Test DNS Resolution (using debug container)
```bash
# Attach ephemeral container and test DNS
kubectl debug <pod> -it --image=busybox -- nslookup <service>
kubectl debug <pod> -it --image=nicolaka/netshoot -- dig <service>.<namespace>.svc.cluster.local
```

### DNS Name Formats
| Format | Scope |
|--------|-------|
| `<service>` | Same namespace |
| `<service>.<namespace>` | Cross-namespace |
| `<service>.<namespace>.svc.cluster.local` | Fully qualified |
| `<pod-ip-dashed>.<namespace>.pod.cluster.local` | Pod DNS |

### Check CoreDNS Health
```
kubectl_get(resourceType="pods", namespace="kube-system", labelSelector="k8s-app=kube-dns")
kubectl_logs(resourceType="pod", name="coredns-xxx", namespace="kube-system")
```

## Connectivity Testing

### From Debug Container
```bash
# TCP connection test
kubectl debug <pod> -it --image=busybox -- nc -zv <host> <port>

# HTTP test
kubectl debug <pod> -it --image=curlimages/curl -- curl -v http://<service>:<port>/health

# Full network diagnosis
kubectl debug <pod> -it --image=nicolaka/netshoot -- /bin/bash
# Then inside: tcpdump, traceroute, mtr, iperf, etc.
```

### Using Port Forward for Local Testing
```
port_forward(resourceType="pod", resourceName="<pod>", namespace="<ns>", localPort=8080, targetPort=80)
```

## NetworkPolicy Debugging

### List NetworkPolicies
```
kubectl_get(resourceType="networkpolicies", namespace="<ns>")
kubectl_describe(resourceType="networkpolicy", name="<policy>", namespace="<ns>")
```

### Common Issues
- Default deny blocks all traffic
- Missing egress rules block outbound
- Label selectors don't match

### Test Without NetworkPolicy
Temporarily delete policy to confirm it's the cause:
```
kubectl_get(resourceType="networkpolicy", namespace="<ns>", output="yaml") > backup.yaml
kubectl_delete(resourceType="networkpolicy", name="<policy>", namespace="<ns>")
# Test connectivity
# Restore: kubectl_apply(manifest=<backup>)
```

## Ingress Debugging

### Check Ingress Status
```
kubectl_get(resourceType="ingress", namespace="<ns>")
kubectl_describe(resourceType="ingress", name="<ingress>", namespace="<ns>")
```

### Verify Ingress Controller
```
kubectl_get(resourceType="pods", namespace="ingress-nginx")
kubectl_logs(resourceType="pod", name="ingress-nginx-controller-xxx", namespace="ingress-nginx")
```

### Common Ingress Issues
- No ADDRESS: Ingress controller not processing
- 404: Backend service not found
- 502: Backend pods not healthy
- 503: No endpoints available

## Node-Level Network Debug

Debug from node's perspective using a privileged pod:

```bash
kubectl debug node/<node-name> -it --image=nicolaka/netshoot
# This creates a pod with host networking and mounts node's filesystem at /host
```
