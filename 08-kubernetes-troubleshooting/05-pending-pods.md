# 🔴 Pending Pods — Won't Schedule

Pod is created but stuck in `Pending`. Kubernetes can't find a node to schedule it on.

```bash
kubectl describe pod myapp-xxx -n production | grep -A10 "Events:"
# Events:
#   Warning  FailedScheduling  0/5 nodes are available:
#            2 Insufficient cpu,
#            3 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }

# The message tells you exactly why - read it carefully
```

## Common scheduling failures

```bash
# Issue 1: Insufficient resources - no node has enough CPU/memory
# Check what's available on nodes
kubectl describe nodes | grep -A5 "Allocated resources:"
# CPU Requests  2450m (31%)     CPU Limits  4900m (62%)
# Memory Requests 4096Mi (53%)  Memory Limits 8192Mi (106%)  ← Over-committed!

# Check what the pod is requesting
kubectl get pod myapp-xxx -n production -o jsonpath=\
'{.spec.containers[*].resources.requests}'
```

```bash
# Issue 2: Node selector or affinity not matching
kubectl describe pod myapp-xxx -n production | grep -A5 "Node-Selectors:"

# Node-Selectors: disktype=ssd   ← Looking for nodes with this label

# Check if any node has that label
kubectl get nodes --show-labels | grep disktype
```

```bash
# Issue 3: Taints blocking scheduling
# Check node taints
kubectl describe node <node-name> | grep Taints
# Taints: CriticalAddonsOnly=true:NoSchedule

# Pod needs a toleration:
# tolerations:
#   - key: "CriticalAddonsOnly"
#     operator: "Exists"
#     effect: "NoSchedule"
```

```bash
# Issue 4: PVC not bound (pod waiting for storage)
kubectl get pvc -n production

# NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES

# myapp-storage  Pending   <none>   0          <none>       ← PVC stuck Pending
kubectl describe pvc myapp-storage -n production
# Events: no persistent volumes available for this claim
# and their node affinity rules conflict with the topology
# = StorageClass can't provision in that AZ
```

---

## Quick Recap

| Issue | Symptom in Events | Fix |
|-------|--------------------|-----|
| Insufficient resources | `Insufficient cpu/memory` | Scale nodes, reduce requests, or check over-commitment |
| Node selector/affinity mismatch | `didn't match Node-Selector` | Label the right nodes or fix the selector |
| Taints without tolerations | `untolerated taint` | Add matching `tolerations` to pod spec |
| PVC not bound | PVC stuck `Pending` | Check `StorageClass`, AZ/zone affinity |

**Golden move:** `kubectl describe pod` → `FailedScheduling` event almost always spells out the exact reason (resources, selector, taint, or storage) — read it before guessing.

**Previous:** [← ImagePullBackOff](./04-imagepullbackoff.md) | **Next:** [Init Container Failures →](./06-init-container-failures.md)
