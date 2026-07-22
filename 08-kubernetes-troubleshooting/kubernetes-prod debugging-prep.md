# Kubernetes Interview Prep — Production Thinking Questions

> Goal: These aren't trivia questions. They test whether you've actually run Kubernetes in production and dealt with real failures — not just memorized commands. For each question, know the "surface answer" but aim to give the "production answer."

---

## 1. What happens when a node runs out of memory?

**Surface answer:** Pods get killed.

**Production answer:** The kubelet detects memory pressure and starts **evicting** pods based on **QoS (Quality of Service) classes**:
- `BestEffort` pods (no requests/limits set) get evicted first.
- `Burstable` pods (using more than their requested memory) go next.
- `Guaranteed` pods (requests = limits) are evicted last, only as a last resort.

**Also mention:**
- **PodDisruptionBudgets (PDBs)** — evictions can cascade and take down a service if you're not careful.
- Set up monitoring/alerts on node memory pressure *before* you hit the eviction threshold.

---

## 2. A pod is stuck in `CrashLoopBackOff`. Where do you start?

**Weak answer:** "Check logs, describe the pod, check events." (Just listing tools, no reasoning.)

**Strong answer — a debugging sequence:**
1. Confirm the image exists and can actually be pulled.
2. Check container logs — did it start at all?
3. If it started and then crashed, look at the **exit code** and any recent code/config changes.
4. If an **init container** is failing, remember the main container never even runs.

**Key idea:** Have a mental checklist that rules out entire categories of failure before diving into details.

---

## 3. You have a 100-node cluster. How do you safely drain one node for maintenance?

**Weak answer:** `kubectl drain node-47`

**Strong answer:**
- Check **PodDisruptionBudgets (PDBs)** first — draining without them can violate service availability.
- Know the difference between `drain` (evicts pods) and `cordon` (marks node unschedulable, but doesn't evict).
- Ask: How many replicas does each affected service have? Are there stateful workloads?
- Never drain multiple nodes at once without checking replica spread.

**Real failure example:** A team drained a node without checking PDBs. All 3 replicas of a service happened to sit on the 3 nodes being drained — no PDB meant Kubernetes allowed all 3 to be evicted at once, taking the whole API down.

---

## 4. How do rolling updates work? What can go wrong?

**Surface answer:** Pods get replaced gradually using `maxSurge` and `maxUnavailable`.

**Production answer, add:**
- If new pods fail **readiness checks**, the rollout stalls.
- If `maxUnavailable` is set too high, you risk temporarily losing capacity.
- If health checks are misconfigured, bad pods can still receive live traffic.
- Understand the difference between **readiness probes** (ready to receive traffic) and **liveness probes** (container is alive, restart if not).
- Know your rollback plan and have observability in place during rollouts to catch problems early — before they hit every replica.

---

## 5. What's the difference between a DaemonSet and a Deployment?

**Textbook answer:** DaemonSet = one pod per node. Deployment = N replicas, scheduler decides placement.

**Production answer:**
- **DaemonSet** — for infrastructure-level needs that must run on every node: log collectors, node monitoring agents, CNI networking plugins.
- **Deployment** — for application workloads where you control replica count.
- Mention **tolerations** and **node selectors** used with DaemonSets.
- A DaemonSet doesn't "scale" the way a Deployment does — it just follows the number of nodes.

---

## 6. How does Kubernetes networking work?

This question separates people who understand the underlying layer from people who just copy-paste YAML.

**Cover these three layers:**
- **Pod-to-pod:** Every pod gets its own IP. Pods talk to each other directly, no NAT. Handled by the **CNI plugin**.
- **Service networking:** A `ClusterIP` gives a stable virtual IP. **kube-proxy** sets up iptables or IPVS rules to load-balance traffic across pod IPs.
- **Ingress:** Handles HTTP(S) routing into the cluster. Needs an **ingress controller** to configure an actual load balancer/reverse proxy.

**Bonus points for mentioning:**
- Network policies (controlling pod-to-pod traffic).
- DNS behaves differently inside vs. outside the cluster.
- MTU mismatch issues.
- Pod IPs aren't reachable from outside the cluster without extra configuration.

---

## 7. A service is intermittently timing out. How do you debug it?

Intermittent issues are common in production and where most candidates struggle.

**Build a hypothesis tree:**
- **Specific pods?** Check which backend pods are serving traffic, look for restarts or high error rates on individual instances.
- **Network?** Check for packet loss, MTU mismatches, CNI issues, confirm service endpoints are properly registered.
- **Application?** Look at latency distributions, check if certain request types are slower, verify connection pool settings.
- **Resource contention?** Check for CPU throttling, node CPU/memory pressure, confirm requests/limits are set correctly.

**Also mention:**
- Using `kubectl proxy` to bypass the Service and hit pods directly to isolate the problem.
- Distributed tracing, if the application supports it.

---

## 8. Explain resource requests vs. limits. Why do both exist?

**Simple answer:** Requests are for scheduling. Limits are for runtime.

**Complete answer:**
- **Requests** tell the scheduler how much capacity a pod needs — the scheduler only places it on a node with enough unreserved capacity.
- **Limits** set the max a pod can consume. Exceed the memory limit → the pod is **OOMKilled**. Exceed the CPU limit → the pod gets **throttled** (not killed).

**Also mention:**
- This ties back to **QoS classes** (see Q1).
- Setting limits without requests is risky (scheduling becomes unpredictable).
- CPU limits set too low cause throttling that looks like app slowness — easy to misdiagnose.
- Memory limits set too low can prevent things like a JVM from even starting.

---

## 9. How would you design for high availability in Kubernetes?

**Weak answer:** "Run multiple replicas."

**Strong answer — start with failure domains:**
- Spread replicas across availability zones using **pod topology spread constraints**.
- Set **PodDisruptionBudgets** so voluntary disruptions don't take the whole service down.
- Configure proper health checks so bad pods don't get traffic.
- Use **readiness gates** if startup is slow.
- Set resource requests/limits correctly to avoid "noisy neighbor" problems.

**Go beyond Kubernetes itself:**
- Application-level resilience: connection pooling, retry logic, circuit breakers, graceful degradation.
- Kubernetes handles infrastructure failure — the application still needs to handle transient errors on its own.
- Operational side matters too: monitoring, alerting, runbooks. HA isn't just architecture — it's also how fast you detect and respond to incidents.

---

## 10. What's your process for upgrading a production cluster?

This question reveals whether you've actually upgraded a cluster or just read about it.

**Real answers cover:**
- **Testing:** Upgrade a non-production cluster first. Verify apps still work. Check for deprecated API versions.
- **Timing:** Upgrade during low-traffic windows. Have a rollback plan. Confirm backups actually work.
- **Sequencing:** Control plane first, then nodes. Drain and upgrade nodes gradually, monitoring app health between batches.
- **Communication:** Notify stakeholders, coordinate across teams, have incident response ready in case something breaks.

**Also mention:**
- Kubernetes **version skew policy** — control plane and node versions can differ by at most one minor version.
- Scanning workloads for deprecated/removed API versions before upgrading.

---

## The Big Picture

These questions aren't about memorizing docs — they're about whether you've actually experienced the failure modes:

- Eviction and QoS pressure
- Debugging crash loops methodically
- Draining nodes without taking down a service
- Rollouts failing silently due to bad health checks
- Understanding the network stack, not just the YAML
- Systematic debugging under ambiguity
- Requests vs. limits and their real runtime effects
- HA design that goes beyond "just add replicas"
- Actually having upgraded a cluster before

**Study tip:** For each question above, try to answer out loud in under 90 seconds, and make sure you can back it up with a real (or realistic) example from experience.
