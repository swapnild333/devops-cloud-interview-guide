# 🔧 The Advanced Debugging Toolkit

## kubectl debug — Ephemeral Debug Containers

Kubernetes 1.23+ supports ephemeral debug containers — inject a debug container into a running pod without restarting it:

```bash
# Inject a debug container into a running pod (non-destructive)
kubectl debug -it myapp-xxx \
    --image=nicolaka/netshoot \
    --target=myapp \
    -n production

# This gives you a shell with full network/process tools
# while the main container keeps running undisturbed

# Debug a distroless container (no shell at all)
kubectl debug -it myapp-xxx \
    --image=busybox \
    --target=myapp \
    --share-processes \   # Share the process namespace
    -n production
# Now you can see the distroless app's processes:
# ps aux   ← Lists all processes including main container
# ls -la /proc/1/root/   ← Browse the main container's filesystem via /proc
```

## Copying a Failing Pod with Debug Override

```bash
# Create a copy of the failing pod with the command overridden
# Great for pods that die too fast to exec into
kubectl debug myapp-xxx \
    --copy-to=myapp-debug \
    --container=myapp \
    --image=myregistry.io/myapp:v2.0.0 \
    -- sleep 3600 \
    -n production

# Now exec into the copy and run the real command manually
kubectl exec -it myapp-debug -n production -- sh
# $ /app/server   ← Run the real app and see what happens
```

## Watching Events in Real Time

```bash
# Watch events cluster-wide as they happen
kubectl get events -A --watch \
    --sort-by='.lastTimestamp'


# Filter for Warning events only (Normals are noise)
kubectl get events -A \
    --field-selector type=Warning \
    --sort-by='.lastTimestamp'

# Events for a specific namespace with count
kubectl get events -n production \
    --sort-by='.lastTimestamp' \
    -o custom-columns=\
'TIME:.lastTimestamp,TYPE:.type,REASON:.reason,OBJECT:.involvedObject.name,MESSAGE:.message'
```

## Checking Resource Limits and Actual Usage

```bash
# Compare requested vs actual usage for all pods in a namespace
kubectl top pods -n production

# See what a pod's limits are
kubectl get pod myapp-xxx -n production \
    -o jsonpath='{range .spec.containers[*]}Container: {.name}
  CPU Request: {.resources.requests.cpu}
  CPU Limit: {.resources.limits.cpu}
  Mem Request: {.resources.requests.memory}
  Mem Limit: {.resources.limits.memory}{"\n"}{end}'

# Find pods with no resource limits (dangerous in production)
kubectl get pods -A -o json | jq -r '
    .items[] |
    select(.spec.containers[].resources.limits == null) |
    "\(.metadata.namespace)/\(.metadata.name)"'
```

## Port Forwarding for Local Debugging

```bash
# Access a pod directly (bypass Service and Ingress)
kubectl port-forward pod/myapp-xxx 8080:3000 -n production
# Now curl localhost:8080 to hit the pod directly

# Access a service locally
kubectl port-forward svc/myapp-service 8080:80 -n production

# Access a deployment (random pod)
kubectl port-forward deployment/myapp 8080:3000 -n production

# Useful for:
# - Testing if the app itself is working (vs networking/ingress issue)
# - Accessing internal services (prometheus, argocd, databases)
# - Comparing app response vs what Ingress returns
kubectl port-forward svc/prometheus-server 9090:9090 -n monitoring &
open http://localhost:9090
```

---

## Quick Recap

| Technique | Use it when... | Command anchor |
|-----------|------------------|----------------|
| Ephemeral debug container | Pod is running but has no shell/tools (distroless) | `kubectl debug -it <pod> --image=... --target=<container>` |
| Copy-to debug pod | Pod crashes too fast to exec into | `kubectl debug <pod> --copy-to=<name> -- sleep 3600` |
| Live event watch | You need to catch an intermittent failure as it happens | `kubectl get events -A --watch --field-selector type=Warning` |
| Resource limit audit | Suspect OOM/CPU throttling or missing limits | `kubectl top pods`, jsonpath / `jq` queries |
| Port-forward | Isolate app-layer vs networking/ingress issue | `kubectl port-forward pod/svc/deployment ...` |

**Interview soundbite:** *"I use ephemeral debug containers instead of restarting or modifying the pod spec — it's non-destructive and works even on distroless images with no shell."*

**Previous:** [← Node-Level Issues](./08-node-level-issues.md) | **Next:** [Conclusion →](./10-conclusion.md)
