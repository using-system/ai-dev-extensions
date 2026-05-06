# AKS kubectl Skill Design

## Problem

The user operates a **private AKS cluster** that is not directly accessible via `kubectl`. All kubectl commands must go through `az aks command invoke`, which proxies the command via the Azure control plane.

Without a skill to enforce this, an AI agent will default to running `kubectl` directly, which will silently fail or timeout on a private cluster.

## Design

### Principle: Intercept and Transform

The skill **forbids** direct `kubectl` execution. Every kubectl command is transformed into:

```bash
az aks command invoke \
  --resource-group <RG> \
  --name <CLUSTER> \
  --command "<kubectl command>"
```

### Cluster Identity via Memory

- On first use, the skill asks the user for `resource-group` and `name`
- These are stored in persistent memory (`memory/aks_cluster.md`, type: `reference`)
- Subsequent sessions read from memory — never ask again
- If the user provides new values, memory is updated
- Only resource-group and name are stored — never credentials or tokens

### Mandatory `--file` for Complex Commands

A command is **complex** if it contains any of:
- Pipe (`|`)
- Chaining (`&&`, `||`, `;`)
- Multi-line content
- Here-doc
- Command substitution (`$(...)` or backticks)
- Length exceeding ~120 characters

Complex commands **must** be written to a temporary script file and sent via `--file`:

```bash
cat > /tmp/aks-cmd.sh << 'EOF'
kubectl get pods -A -o json | jq -r '.items[] | select(.status.phase != "Running") | "\(.metadata.namespace)/\(.metadata.name) \(.status.phase)"'
EOF

az aks command invoke -g myRG -n myCluster \
  -c "bash aks-cmd.sh" \
  -f /tmp/aks-cmd.sh
```

Simple commands (single kubectl with no pipes/chaining) use inline `--command`.

### Sending Local Files to the Cluster

The `--file` parameter also serves to **upload local files** (manifests, configs) that the command references:

```bash
az aks command invoke -g myRG -n myCluster \
  -c "kubectl apply -f deployment.yaml" \
  -f deployment.yaml
```

Use `.` to send the entire current directory.

### Anti-Patterns

| Pattern | Why it fails |
|---------|-------------|
| `kubectl get pods` (direct) | Private cluster, no network path |
| Complex command as inline `--command` | Quoting hell, truncation, shell escaping issues |

### Examples

**Simple (inline):**
```bash
az aks command invoke -g myRG -n myCluster -c "kubectl get pods -n default"
az aks command invoke -g myRG -n myCluster -c "kubectl logs deploy/my-app -n prod"
```

**Complex (file):**
```bash
cat > /tmp/aks-cmd.sh << 'EOF'
kubectl get pods -A -o json | jq -r '.items[] | select(.status.phase != "Running") | "\(.metadata.namespace)/\(.metadata.name) \(.status.phase)"'
EOF

az aks command invoke -g myRG -n myCluster \
  -c "bash aks-cmd.sh" \
  -f /tmp/aks-cmd.sh
```

**Apply local manifest:**
```bash
az aks command invoke -g myRG -n myCluster \
  -c "kubectl apply -f deployment.yaml" \
  -f deployment.yaml
```

## Scope

- kubectl only (no helm)
- Single cluster (one resource-group/name pair)
- Skill lives in `skills/aks-kubectl/SKILL.md`
