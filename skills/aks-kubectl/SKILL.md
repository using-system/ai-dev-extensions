---
name: aks-kubectl
description: "Use when running kubectl commands against an AKS cluster - enforces az aks command invoke for private cluster access, requires --file for complex commands, and persists cluster identity in memory"
---

# AKS kubectl via az aks command invoke

## Overview

All kubectl commands MUST be executed through `az aks command invoke` because the target AKS cluster is private and not directly reachable. This skill intercepts any attempt to run `kubectl` directly and transforms it into the correct `az aks command invoke` call.

## When to Use

- Running any kubectl command (get, describe, logs, apply, delete, exec, etc.)
- Deploying manifests or configs to the AKS cluster
- Debugging pods, services, or workloads on the cluster
- Any operation that would normally require direct kubectl access

## Rules

### 0. Never run kubectl directly

kubectl will silently fail or timeout on a private cluster. Every kubectl command MUST go through `az aks command invoke`.

```bash
# BAD - will timeout on private cluster
kubectl get pods
kubectl apply -f deployment.yaml
kubectl logs deploy/my-app

# GOOD - routed through Azure control plane
az aks command invoke -g <RG> -n <CLUSTER> -c "kubectl get pods"
```

### 1. Cluster identity from memory

The `--resource-group` and `--name` parameters identify the target cluster.

**First use (no memory found):** Ask the user:
> "I need your AKS cluster details to run kubectl commands. What is the resource-group and cluster name?"

Then store in persistent memory (`memory/aks_cluster.md`, type: `reference`).

**Subsequent uses:** Read from memory. Never ask again unless the user explicitly provides new values.

**If the user provides new values:** Update the memory file.

### 2. Simple commands use inline --command

A command is **simple** if it has:
- No pipe (`|`)
- No chaining (`&&`, `||`, `;`)
- No command substitution (`$(...)` or backticks)
- No multi-line content or here-doc
- Length under ~120 characters

Simple commands go directly in `--command`:

```bash
# List pods
az aks command invoke -g <RG> -n <CLUSTER> -c "kubectl get pods -n default"

# Pod logs
az aks command invoke -g <RG> -n <CLUSTER> -c "kubectl logs deploy/my-app -n prod"

# Describe a service
az aks command invoke -g <RG> -n <CLUSTER> -c "kubectl describe svc my-service -n prod"

# Delete a pod
az aks command invoke -g <RG> -n <CLUSTER> -c "kubectl delete pod my-pod -n default"
```

### 3. Complex commands MUST use --file

A command is **complex** if it contains ANY of:
- Pipe (`|`)
- Chaining (`&&`, `||`, `;`)
- Multi-line content
- Here-doc
- Command substitution (`$(...)` or backticks)
- Length exceeding ~120 characters

Complex commands MUST be written to a temporary script file and sent via `--file`:

**Important:** `--file` uploads the file to the command pod's **working directory**, so `--command` must reference the filename only (not the local path).

```bash
# Step 1: Write the script locally
cat > /tmp/aks-cmd.sh << 'EOF'
kubectl get pods -A -o json | jq -r '.items[] | select(.status.phase != "Running") | "\(.metadata.namespace)/\(.metadata.name) \(.status.phase)"'
EOF

# Step 2: Send it via --file, reference by filename in --command
az aks command invoke -g <RG> -n <CLUSTER> \
  -c "bash aks-cmd.sh" \
  -f /tmp/aks-cmd.sh
```

Another example with chaining:

```bash
cat > /tmp/aks-cmd.sh << 'EOF'
kubectl get nodes -o wide && kubectl get pods -A --field-selector=status.phase!=Running
EOF

az aks command invoke -g <RG> -n <CLUSTER> \
  -c "bash aks-cmd.sh" \
  -f /tmp/aks-cmd.sh
```

### 4. Sending local files to the cluster

When a kubectl command references a local file (manifest, configmap, secret YAML), use `--file` to upload it:

```bash
# Apply a single manifest
az aks command invoke -g <RG> -n <CLUSTER> \
  -c "kubectl apply -f deployment.yaml" \
  -f deployment.yaml

# Apply multiple files — send the whole directory
az aks command invoke -g <RG> -n <CLUSTER> \
  -c "kubectl apply -f ." \
  -f .
```

**Important:** The file referenced in `--command` must match what is sent via `--file`. The file is uploaded to the command pod's working directory.

### 5. Combining script + local files

When a complex command also needs local files, send both the script AND the files. Use `.` to send the current directory which includes everything:

```bash
cat > /tmp/apply-and-check.sh << 'EOF'
kubectl apply -f deployment.yaml && kubectl rollout status deployment/my-app -n prod
EOF

cp /tmp/apply-and-check.sh .
az aks command invoke -g <RG> -n <CLUSTER> \
  -c "bash apply-and-check.sh" \
  -f .
```

## Anti-Patterns

| Pattern | Problem |
|---------|---------|
| `kubectl get pods` | Direct kubectl — will timeout on private cluster |
| `az aks command invoke -c "kubectl get pods -o json \| jq '.items[]'"` | Complex command inline — quoting issues, shell escaping breaks |
| Forgetting `--file` when command references a local file | File not found on the command pod |
| Hardcoding resource-group/name instead of reading from memory | Breaks portability across sessions |

## Quick Reference

```bash
# Simple command pattern
az aks command invoke -g <RG> -n <CLUSTER> -c "<kubectl command>"

# Complex command pattern (file — reference by filename, not local path)
cat > /tmp/aks-cmd.sh << 'EOF'
<complex command here>
EOF
az aks command invoke -g <RG> -n <CLUSTER> -c "bash aks-cmd.sh" -f /tmp/aks-cmd.sh

# Apply local manifest pattern
az aks command invoke -g <RG> -n <CLUSTER> -c "kubectl apply -f <file>" -f <file>
```
