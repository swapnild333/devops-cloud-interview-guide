# 🔴 Init Container Failures

Init containers run before your main container and must complete successfully. If they fail, your pod never starts.

```bash
# Init container visible in pod status
kubectl get pod myapp-xxx -n production
# NAME        READY   STATUS                  RESTARTS   AGE
# myapp-xxx   0/1     Init:CrashLoopBackOff   3          5m

# Get init container logs
kubectl logs myapp-xxx -n production -c init-check-db

# Error: connection refused - postgres:5432
```

```bash
# List all containers (including init containers)
kubectl describe pod myapp-xxx -n production | grep -E "Init Containers:|Containers:" -A20
```

---

## Quick Recap

- **Symptom:** Pod status shows `Init:CrashLoopBackOff`, `Init:Error`, or `Init:N/M` (M init containers, N completed).
- **Key detail:** Use `-c <init-container-name>` with `kubectl logs` — without it, you'll get the main container's (empty) logs.
- **Common root causes:** init container waiting on a dependency that isn't ready (e.g., DB, another service), misconfigured wait/check script, missing env vars or secrets.
- **List everything:** `kubectl describe pod` with grep on `Init Containers:` and `Containers:` shows the full container topology at a glance.

**Previous:** [← Pending Pods](./05-pending-pods.md) | **Next:** [Networking Issues →](./07-networking-issues.md)
