# 🔴 CrashLoopBackOff

The most common error. Your container starts, crashes, Kubernetes restarts it, it crashes again. After enough restarts, K8s backs off the restart timing exponentially, hence `CrashLoopBackOff`.

```bash
# Get the current status
kubectl get pod myapp-xxx -n production
# NAME         READY   STATUS             RESTARTS   AGE
# myapp-xxx    0/1     CrashLoopBackOff   8          12m


# Step 1: Read events - often tells you everything
kubectl describe pod myapp-xxx -n production

# Look for the Events section at the bottom:
# Events:
#   Warning  BackOff   2m    kubelet  Back-off restarting failed container

# Step 2: Get logs from the CURRENT crash
kubectl logs myapp-xxx -n production

# Step 3: Get logs from the PREVIOUS crash (often more useful)
kubectl logs myapp-xxx -n production --previous

# Step 4: If logs are empty (container dies before logging), check exit code
kubectl describe pod myapp-xxx -n production | grep -A5 "Last State:"

# Last State:  Terminated
#   Reason:    Error
#   Exit Code: 1        ← Non-zero = error. 137 = OOMKilled. 1 = app error. 2 = misuse.
#   Started:   Mon, 01 Jan 2026 10:00:00 +0000
#   Finished:  Mon, 01 Jan 2026 10:00:02 +0000  ← Died in 2 seconds
```

## Common CrashLoopBackOff causes by exit code

```
Exit Code  Meaning                          Investigation
──────────────────────────────────────────────────────────────────
0          Successful exit (container done) Wrong CMD - app treats as job, not server
1          Application error                App logs → missing config, bug, startup failure
2          Misuse of shell command          Check CMD/ENTRYPOINT syntax
137        SIGKILL (OOMKilled or kill -9)   Check memory limits → increase or fix leak
139        Segfault (SIGSEGV)               Application bug or corrupt binary
143        Graceful SIGTERM not handled     Liveness probe killing healthy container
```

## Debugging a container that dies too fast to log

```bash
# Override the command to keep it alive - inspect the environment
kubectl run debug-myapp \
    --image=myregistry.io/myapp:v2.0.0 \
    --restart=Never \
    --command -- sleep 3600

# Shell in and run the actual startup command manually
kubectl exec -it debug-myapp -- sh

# Inside the container:
# $ /app/server --config /etc/config/app.yaml
# Error: config file not found: /etc/config/app.yaml   ← Found it!

# Check if required env vars are present
kubectl exec -it debug-myapp -- env | grep -E "DB_HOST|API_KEY|PORT"

# Check if the ConfigMap/Secret is actually mounted
kubectl exec -it debug-myapp -- ls -la /etc/config/

# Clean up
kubectl delete pod debug-myapp
```

---

## Quick Recap

- **Symptom:** `STATUS` shows `CrashLoopBackOff`, `RESTARTS` climbing.
- **First move:** `kubectl describe pod` → read Events, then `Last State` for exit code.
- **Exit code cheat sheet:** `0`=wrong CMD, `1`=app error, `2`=shell misuse, `137`=OOMKilled, `139`=segfault, `143`=SIGTERM not handled/liveness probe killing it.
- **If logs are empty:** spin up a debug pod with `sleep 3600` override and exec in to run the startup command manually.
- **Don't forget:** `kubectl logs --previous` for the crash *before* the current one — often more informative than the latest.

**Previous:** [← Debugging Mental Model](./01-debugging-mental-model.md) | **Next:** [OOMKilled →](./03-oomkilled.md)
