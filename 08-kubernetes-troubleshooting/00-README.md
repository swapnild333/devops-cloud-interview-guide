# Kubernetes Troubleshooting: A Senior Engineer's Guide

> Debugging pods, nodes, and networking — the kubectl commands and mental models that cut your debugging time from hours to minutes.

Source: *ATNO For DevOps Engineers*, Mar 29, 2026 (12 min read)

## 📚 Contents

1. [Debugging Mental Model](./01-debugging-mental-model.md)
2. [CrashLoopBackOff](./02-crashloopbackoff.md)
3. [OOMKilled](./03-oomkilled.md)
4. [ImagePullBackOff / ErrImagePull](./04-imagepullbackoff.md)
5. [Pending Pods — Won't Schedule](./05-pending-pods.md)
6. [Init Container Failures](./06-init-container-failures.md)
7. [Networking Issues](./07-networking-issues.md)
   - DNS Issues
   - Service Not Routing Traffic
   - Network Policy Blocking Traffic
   - kube-proxy / iptables Issues
8. [Node-Level Issues](./08-node-level-issues.md)
   - Node NotReady
   - Node Disk Pressure
   - Node Resource Exhaustion Debugging
9. [Advanced Debugging Toolkit](./09-advanced-debugging-toolkit.md)
   - kubectl debug — Ephemeral Debug Containers
   - Copying a Failing Pod with Debug Override
   - Watching Events in Real Time
   - Checking Resource Limits and Actual Usage
   - Port Forwarding for Local Debugging
10. [Conclusion](./10-conclusion.md)

## 🧠 Quick Reference — First 3 Commands

```bash
kubectl get pods -n <namespace>                       # What's the status?
kubectl describe pod <pod-name> -n <namespace>        # What happened? (events!)
kubectl logs <pod-name> -n <namespace> --tail=100     # What did the app say?
```

**Golden rule: read the events first.** Most engineers go straight to logs and miss the obvious.

## 🗂️ How to use this repo for interview prep

- Each file is self-contained — read one topic per sitting.
- Every file ends with a **Quick Recap** section (added for revision) summarizing the key commands/causes at a glance.
- Try covering the "Fix" sections and quizzing yourself from the symptom → command → root cause.
