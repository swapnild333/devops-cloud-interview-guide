# 🔴 OOMKilled

The container exceeded its memory limit. Kubernetes killed it with `SIGKILL` (exit code 137). Unlike CPU throttling (which just slows things down), memory has a hard limit — exceed it and the pod dies.

```bash
# Confirm it's OOMKilled
kubectl describe pod myapp-xxx -n production | grep -A5 "Last State"
# Last State:  Terminated
#   Reason:    OOMKilled      ← There it is
#   Exit Code: 137

# See what limits are currently set
kubectl get pod myapp-xxx -n production -o jsonpath=\
'{.spec.containers[0].resources}' | jq

# Check actual memory usage over time
kubectl top pod myapp-xxx -n production
# If metrics-server is installed:
# NAME        CPU(cores)   MEMORY(bytes)

# myapp-xxx   245m         248Mi         ← Close to or exceeding your limit

# Watch memory across all pods in namespace
kubectl top pods -n production --sort-by=memory
```

## Fixing OOMKilled

```yaml
# Option 1: Increase memory limit (quick fix)
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"    # Was 256Mi - doubled it
```

```bash
# Option 2: If it's a memory leak, find the leak
# Add heap dump on OOM for JVM apps:
env:
  - name: JAVA_OPTS
    value: "-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof"
# Retrieve the heap dump before pod restarts
kubectl cp myapp-xxx:/tmp/heapdump.hprof ./heapdump.hprof -n production
# For Node.js - check for common memory leaks
# Add --max-old-space-size and enable heap profiling:
# node --max-old-space-size=400 --inspect server.js

# Check if the issue is a specific endpoint causing memory spike
# Look for memory growing after specific requests in logs
kubectl logs myapp-xxx -n production | grep -E "heap|memory|OOM"
```

---

## Quick Recap

- **Symptom:** Exit code `137`, `Reason: OOMKilled` in `Last State`.
- **Confirm:** `kubectl describe pod` + `kubectl top pod` (needs metrics-server).
- **Quick fix:** Bump `resources.limits.memory`.
- **Real fix (leak):** JVM → heap dump on OOM + `kubectl cp` the `.hprof` file; Node.js → `--max-old-space-size` + heap profiling.
- **Investigate:** Correlate memory growth with specific endpoints/requests in logs.
- **Remember:** Memory is a hard limit (kill), unlike CPU which is throttled, not killed.

**Previous:** [← CrashLoopBackOff](./02-crashloopbackoff.md) | **Next:** [ImagePullBackOff →](./04-imagepullbackoff.md)
