# 🖥️ Node-Level Issues

## Node NotReady

```bash
kubectl get nodes
# NAME         STATUS     ROLES    AGE   VERSION
# node-1       Ready      worker   10d   v1.29.0
# node-2       NotReady   worker   10d   v1.29.0    ← Problem

# Investigate the NotReady node
kubectl describe node node-2 | grep -A10 "Conditions:"

# Conditions:
#   Type              Status  Reason
#   MemoryPressure    False   ...
#   DiskPressure      True    KubeletHasDiskPressure   ← Disk full!
#   PIDPressure       False   ...
#   Ready             False   KubeletNotReady
```

Conditions to watch:
- **DiskPressure** → Node disk full. Clean up images: `docker image prune -af`
- **MemoryPressure** → Node RAM exhausted. Too many pods, or a memory leak on the node
- **PIDPressure** → Too many processes. Usually a container fork-bombing

```bash
# Get events for the node
kubectl get events -n default --field-selector involvedObject.name=node-2 \
    --sort-by='.lastTimestamp'

# Check kubelet status (if you have node access via SSM/bastion)
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager
```

## Node Disk Pressure

```bash
# Find what's eating disk space on the node
# Using kubectl debug to access the node
kubectl debug node/node-2 -it --image=ubuntu -- bash

# Inside node shell (mounted at /host):
df -h /host
du -sh /host/var/lib/docker/overlay2/* | sort -rh | head -20

# Clean up unused container images (safe - only removes unused ones)
crictl rmi --prune    # For containerd

# OR
docker image prune -af  # For Docker

# Check if logs are filling disk
du -sh /host/var/log/pods/* | sort -rh | head -10
```

## Node Resource Exhaustion Debugging

```bash
# See how resources are allocated across all nodes
kubectl describe nodes | grep -A5 "Allocated resources"

# Which pods are on a specific node
kubectl get pods -A --field-selector spec.nodeName=node-2 \
    -o custom-columns=\
'NAMESPACE:.metadata.namespace,NAME:.metadata.name,CPU:.spec.containers[*].resources.requests.cpu,MEM:.spec.containers[*].resources.requests.memory'

# Top pods on a specific node
kubectl top pods -A --field-selector spec.nodeName=node-2 \
    --sort-by=memory

# Check if node has any system pods eating resources
kubectl get pods -n kube-system --field-selector spec.nodeName=node-2
```

---

## Quick Recap

| Condition | Meaning | Fix |
|-----------|---------|-----|
| `DiskPressure=True` | Node disk full | Prune images (`crictl rmi --prune` / `docker image prune -af`), check `/var/log/pods` |
| `MemoryPressure=True` | Node RAM exhausted | Reduce pod density, fix memory leak, add nodes |
| `PIDPressure=True` | Too many processes | Investigate fork-bombing container, set pod PID limits |
| `Ready=False` | kubelet unhealthy overall | `systemctl status kubelet`, `journalctl -u kubelet` |

**Node access pattern:** `kubectl debug node/<node-name> -it --image=ubuntu -- bash` mounts the host filesystem at `/host` — no SSH needed.

**Previous:** [← Networking Issues](./07-networking-issues.md) | **Next:** [Advanced Debugging Toolkit →](./09-advanced-debugging-toolkit.md)
