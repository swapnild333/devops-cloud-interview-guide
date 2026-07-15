# 🧠 The Debugging Mental Model

Before any command, the framework:

```
1. Observe    → What exactly is wrong? (status, events, logs)
2. Locate     → Where is the failure? (pod, node, network, control plane)
3. Understand → Why is it failing? (root cause, not symptom)
4. Fix        → Address root cause, not symptom
5. Verify     → Confirm the fix, check for side effects
```

And the golden rule: **read the events first**. Kubernetes events tell you exactly what the system tried to do and what went wrong. Most engineers go straight to logs and miss the obvious.

## Your first three commands in any debugging session

```bash
kubectl get pods -n <namespace>                       # What's the status?
kubectl describe pod <pod-name> -n <namespace>        # What happened? (events!)
kubectl logs <pod-name> -n <namespace> --tail=100     # What did the app say?
```

---

## Quick Recap

| Step | Question | Command family |
|------|----------|-----------------|
| Observe | What's wrong? | `kubectl get pods` |
| Locate | Where's the failure? | `kubectl describe pod/node` |
| Understand | Why? | `kubectl logs`, events |
| Fix | Root cause | edit manifest / config |
| Verify | Did it work? | `kubectl get`, `kubectl top` |

**Interview tip:** When asked "how do you debug a failing pod," lead with this 5-step framework before naming any commands — it shows structured thinking, not just memorized commands.

**Next:** [CrashLoopBackOff →](./02-crashloopbackoff.md)
