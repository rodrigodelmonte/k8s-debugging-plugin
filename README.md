# Kubernetes Debugging Plugin for Claude Code

A Claude Code plugin providing systematic Kubernetes debugging and troubleshooting workflows using MCP kubernetes tools.

## Installation

### 1. Install Kubernetes MCP Server (Required)

```bash
claude mcp add kubernetes --scope user -- npx mcp-server-kubernetes
```

**Requirements:**
- Kubernetes cluster access via kubectl
- kubeconfig at `~/.kube/config` or `KUBECONFIG` env var

### 2. Install the Plugin

```bash
# Add the marketplace
/plugin marketplace add rodrigodelmonte/k8s-debugging-plugin

# Install the plugin
/plugin install k8s-debugging-plugin
```

## Features

| Feature | Description |
|---------|-------------|
| **Pod Debugging** | CrashLoopBackOff, ImagePullBackOff, Pending states |
| **Service/Network** | DNS resolution, endpoints, connectivity |
| **Deployments** | Rollout status, history, rollback |
| **Nodes** | Health checks, cordon/drain/uncordon |
| **Resources** | OOMKilled, CPU throttling, quotas |
| **Ephemeral Containers** | Debug distroless/minimal images |

## Usage

The skill triggers automatically when you ask about:
- "Why is my pod failing?"
- "Debug CrashLoopBackOff"
- "Service is unreachable"
- "Deployment not rolling out"
- "Node is NotReady"

## MCP Tools Used

- `kubectl_get` - List resources, check status
- `kubectl_describe` - Detailed info, events, conditions
- `kubectl_logs` - Container logs, application errors
- `exec_in_pod` - Run commands inside containers
- `kubectl_rollout` - Deployment rollout management
- `node_management` - Cordon/drain/uncordon nodes

## License

MIT
