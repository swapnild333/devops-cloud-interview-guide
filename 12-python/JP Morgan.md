# Senior DevOps / Azure / AKS Scenario-Based Interview Q&A

A deep-dive answer guide for real-world architecture, troubleshooting, and design questions asked in senior DevOps / Cloud / SRE interviews.

---

## Table of Contents
1. [AKS random health check failures](#1-aks-random-health-check-failures)
2. [Canary deployment — partial 502s](#2-canary-deployment--partial-502s)
3. [CI/CD pipeline takes 40 minutes](#3-cicd-pipeline-takes-40-minutes)
4. [High CPU, clean logs](#4-high-cpu-clean-logs)
5. [HA logging system for 100+ microservices](#5-ha-logging-system-for-100-microservices)
6. [Works internally, 403 externally](#6-works-internally-403-externally)
7. [Secure secret rotation in Azure DevOps](#7-secure-secret-rotation-in-azure-devops)
8. [App Gateway + WAF for banking app](#8-app-gateway--waf-for-banking-app)
9. [Intermittent DNS issues during Azure deployment](#9-intermittent-dns-issues-during-azure-deployment)
10. [10-second delays every 15 minutes on AKS](#10-10-second-delays-every-15-minutes-on-aks)
11. [Jenkins failing randomly at artifact upload](#11-jenkins-failing-randomly-at-artifact-upload)
12. [Automated rollback strategy in Kubernetes](#12-automated-rollback-strategy-in-kubernetes)
13. [Cost-optimized nightly reporting app, 3-year log retention](#13-cost-optimized-nightly-reporting-app-3-year-log-retention)
14. [Zero-downtime database migrations](#14-zero-downtime-database-migrations)
15. [Disaster recovery for stateful containerized apps](#15-disaster-recovery-for-stateful-containerized-apps)
16. [Azure Function throttling](#16-azure-function-throttling)
17. [Blue/green deployment with Terraform + rollback](#17-bluegreen-deployment-with-terraform--rollback)
18. [End-to-end SLA monitoring for a payments pipeline](#18-end-to-end-sla-monitoring-for-a-payments-pipeline)
19. [Scaling: compute-intensive vs I/O-intensive](#19-scaling-compute-intensive-vs-io-intensive)
20. [Pipeline blocked, stakeholders unreachable](#20-pipeline-blocked-stakeholders-unreachable)

---

## 1. AKS random health check failures

**Approach — work outward from the pod:**

1. **Check the probe config first** — liveness/readiness probes with too-short `timeoutSeconds` or `periodSeconds` will fail under momentary load even when the app is healthy.
   ```bash
   kubectl describe pod <pod> -n <namespace>
   ```
   Look at `Liveness`/`Readiness` sections and recent `Events` for `Unhealthy` messages with response times.

2. **Check resource pressure** — a pod under CPU throttling (hitting its CPU limit) can respond to the health endpoint slowly enough to fail the probe.
   ```bash
   kubectl top pod <pod> -n <namespace>
   ```
   Cross-check against `resources.limits.cpu` in the manifest — if usage consistently hits the limit, that's your answer.

3. **Check node-level issues** — noisy neighbor pods, node CPU/memory pressure, or node network plugin (CNI) issues affecting only some pods.
   ```bash
   kubectl describe node <node>
   kubectl get events --field-selector involvedObject.kind=Node
   ```

4. **Check the health endpoint itself** — does it do a DB ping or downstream call? If the health check depends on a flaky dependency, the app is "healthy" but the check isn't.

5. **Check kube-proxy / Service / Ingress layer** — random 502s but not consistent to one pod usually means Service endpoint list is stale or the Ingress controller is routing to a pod mid-restart.

6. **Application Insights / Azure Monitor for Containers** — correlate probe failures with GC pauses, thread pool exhaustion, or slow downstream calls at the exact failure timestamps.

**Common real root cause:** liveness probe timeout too aggressive (e.g., 1s) combined with periodic GC pauses in a JVM/.NET app — the fix is usually tuning `initialDelaySeconds`/`timeoutSeconds` and separating liveness (process alive) from readiness (dependency-ready) checks properly.

---

## 2. Canary deployment — partial 502s

**Approach:**

1. **Confirm it's actually the canary, not random** — check Ingress/Service Mesh (Istio/Linkerd) traffic split percentage vs error rate correlation.
   ```bash
   kubectl get pods -l version=canary
   kubectl logs <canary-pod> --tail=100
   ```

2. **Check canary pod readiness** — a 502 from the LB/Ingress usually means the upstream (canary pod) refused or reset the connection, not that the app returned an error code.
   ```bash
   kubectl describe pod <canary-pod>
   ```
   Look for `CrashLoopBackOff`, OOMKilled, or readiness probe failures — 502s often mean the canary pods aren't actually ready to serve traffic yet but were added to the endpoint list too early.

3. **Check for a version mismatch on a shared dependency** — canary pod pointing to a new schema/API contract that the DB or downstream service doesn't support yet (classic backward-compatibility break).

4. **Check resource limits on canary pods** — if canary is deployed with fewer replicas/resources than stable, it can get overwhelmed by even a 50% traffic split, causing connection resets under load.

5. **Check Ingress/Gateway timeout settings** — canary pod may have a slower cold start (JIT warmup, connection pool not primed) causing timeouts that surface as 502 at the LB.

**Fix path:** Roll back canary traffic weight to 0% immediately (mitigate first), then reproduce in staging with the same traffic split to isolate readiness-probe timing vs actual application bug before re-attempting.

---

## 3. CI/CD pipeline takes 40 minutes

**Approach — profile before optimizing:**

1. **Get stage-by-stage timing** — Azure DevOps/Jenkins/GitHub Actions all show per-stage duration. Identify the actual bottleneck instead of guessing.

2. **Common culprits, in order of likelihood:**
   - **No dependency caching** — `npm install`/`pip install`/Maven downloading from scratch every run. Fix: cache `node_modules`, `.m2`, or use a private artifact proxy (Azure Artifacts, Nexus).
   - **Docker image builds without layer caching** — rebuilding from scratch each time. Fix: use BuildKit cache mounts or a registry-based cache (`--cache-from`).
   - **Sequential steps that could run in parallel** — e.g., unit tests, lint, and security scans running one after another instead of as parallel jobs/matrix builds.
   - **Oversized test suites running every time** — no test splitting/sharding, or integration tests running on every commit instead of just on merge to main.
   - **Cold agent/runner startup** — spinning up a fresh VM each time instead of using a warm agent pool or self-hosted runners.
   - **Large artifact transfer** — uploading full build output instead of only what's needed for deploy.

3. **Quantify the fix impact before shipping it** — e.g., "dependency caching cut install from 8 min to 45 sec" is what you present to stakeholders.

**Real-world target:** Getting a 40-min pipeline to under 10 min usually comes from (a) caching dependencies, (b) parallelizing independent stages, and (c) using pre-baked build images instead of installing tools fresh each run — not from "buying bigger runners."

---

## 4. High CPU, clean logs

**Approach:**

1. **Confirm it's actually the app, not the sidecar** — `kubectl top pod` shows total; check per-container:
   ```bash
   kubectl top pod <pod> --containers
   ```
   Sometimes it's a service mesh sidecar (Envoy/Istio) or a logging agent consuming CPU, not the app itself.

2. **Get a thread/stack dump** — clean logs mean the issue isn't being logged, so get direct visibility:
   - Java: `kubectl exec <pod> -- jstack <pid>`
   - .NET: `dotnet-trace` or `dotnet-dump`
   - Node.js: `kubectl exec <pod> -- kill -USR1 <pid>` (triggers inspector) or use `clinic.js`

3. **Check for a runaway loop or retry storm** — a common silent cause: a background job/reconciler stuck retrying against a failing dependency in a tight loop with no backoff, burning CPU without ever logging an "error" because it's caught and swallowed.

4. **Check GC behavior** — excessive garbage collection (memory pressure causing constant GC cycles) shows as high CPU with no application-level errors logged.

5. **Check for cron/job overlap** — a scheduled job that didn't finish before the next run started, now running two instances concurrently.

6. **eBPF/profiling tools** — `kubectl exec` in with `perf` or use Azure Monitor's Profiler / Pyroscope for continuous profiling to catch it without needing to reproduce live.

**Real root cause pattern:** Silent infinite retry loops (e.g., against a rate-limited or down dependency) are the most common cause of "high CPU, no errors" — the exception is caught and handled, so it never surfaces in logs, but the retry-with-no-backoff burns cycles constantly.

---

## 5. HA logging system for 100+ microservices

**Architecture:**

```
Microservices (100+, 3 regions)
        │
        ▼
  Fluent Bit (DaemonSet, per-node sidecar)
        │  — lightweight, low resource footprint, buffers on disk during outages
        ▼
  Regional Kafka cluster (buffer/decouple layer)
        │  — absorbs spikes, survives downstream outages, replay capability
        ▼
  Log processing (Logstash / Vector) — parse, enrich, redact PII
        │
        ▼
  Centralized store: Elasticsearch (hot tier) + object storage (cold tier, e.g. Azure Blob)
        │
        ▼
  Kibana / Grafana for visualization + alerting (Alertmanager / Azure Monitor alerts)
```

**Key design decisions:**
- **Per-region Kafka + per-region ES hot tier** — avoids cross-region latency/cost for every log line; only aggregate at query time via a federated view or ship rollups cross-region.
- **Fluent Bit over Fluentd/Logstash at the edge** — much lower memory footprint per node, critical at 100+ services scale.
- **Kafka as a buffer, not just a pipe** — decouples producers from consumers so an ES outage doesn't cause log loss; also enables replay for reprocessing.
- **Index lifecycle management (ILM)** — hot (7 days, fast SSD) → warm (30 days) → cold/frozen (long-term, cheap object storage) to control cost.
- **Structured logging enforced at the app level** (JSON with trace IDs) — enables correlation across microservices via a shared `trace_id`/`correlation_id`, tied into distributed tracing (OpenTelemetry).
- **PII redaction at the processing layer**, not at the source — centralizes compliance logic instead of trusting every team's service to redact correctly.
- **Multi-region resilience** — if one region's ES cluster goes down, Kafka buffers locally and logs replay once it recovers; don't make cross-region log shipping synchronous.

**Alternative for Azure-native shops:** Azure Monitor / Log Analytics Workspace with Container Insights, if the org prefers managed-first — trades some flexibility for far less operational overhead than running ELK yourself.

---

## 6. Works internally, 403 externally

**Approach — isolate what changes between internal and external requests:**

1. **Check the WAF/Application Gateway rules** — 403 is a classic WAF signature match false-positive (e.g., blocking a legitimate query string pattern that looks like SQLi/XSS).
   ```bash
   # Azure: check WAF logs
   az monitor log-analytics query --workspace <id> \
     --analytics-query "AzureDiagnostics | where Category == 'ApplicationGatewayFirewallLog' | where action_s == 'Blocked'"
   ```

2. **Check if internal traffic bypasses the WAF entirely** — internal users often hit the app via a private endpoint or internal load balancer that skips App Gateway/WAF, while external users go through the public-facing WAF layer. This is the #1 cause of this exact symptom.

3. **Check auth/token differences** — external users might be hitting a different auth flow (e.g., B2C vs internal AD) that's misconfigured, returning 403 from the app itself rather than infrastructure.

4. **Check IP allow-listing / NSG rules** — a Network Security Group or App Service access restriction may allow the internal corporate IP range but not external/public IPs.

5. **Check CORS or Origin header handling** — if external users are on a different domain/origin, some apps 403 on unexpected Origin headers as a security measure.

**Isolation technique:** `curl -v` the endpoint from both an internal machine and an external one (or use a tool like a cloud shell with a public IP) with identical headers, then diff the actual HTTP request being sent — the difference tells you exactly which layer (WAF, NSG, App Gateway, app-level auth) is responsible.

---

## 7. Secure secret rotation in Azure DevOps pipelines

**Approach:**

1. **Never store secrets in pipeline YAML or variable groups directly** — use **Azure Key Vault** as the source of truth, linked via a Key Vault-backed variable group in Azure DevOps.

2. **Pipeline references secrets at runtime, not build time:**
   ```yaml
   variables:
   - group: 'kv-linked-secrets'  # Variable group backed by Key Vault
   
   steps:
   - task: AzureKeyVault@2
     inputs:
       azureSubscription: 'service-connection-name'
       KeyVaultName: 'my-keyvault'
       SecretsFilter: '*'
   ```

3. **Automate rotation with Key Vault's native rotation policy** (for supported secret types) or an Azure Function triggered on a schedule that rotates the secret and updates the Key Vault version — apps/pipelines always pull the latest version, never a hardcoded one.

4. **Use Managed Identity for the pipeline's service connection** — eliminates a static service principal secret entirely; the pipeline authenticates to Azure via workload identity federation (OIDC) instead of a stored client secret.

5. **Short-lived tokens over long-lived secrets** — for anything that supports it (e.g., DB access), use Azure AD token-based auth instead of static connection strings.

6. **Audit and alert** — enable Key Vault diagnostic logging to Log Analytics, alert on unexpected `SecretGet` access patterns or failed rotations.

**Key principle to state in interview:** the pipeline should never *own* a secret's lifecycle — it should only *reference* Key Vault at runtime, so rotation happens independently and doesn't require a pipeline change or redeploy.

---

## 8. App Gateway + WAF for banking app

**Architecture and reasoning:**

```
Internet
   │
   ▼
Azure Application Gateway (WAF_v2 SKU)
   │  — TLS termination, OWASP Core Rule Set 3.2+, custom rules
   │  — Bot protection, geo-filtering
   ▼
Backend pool: AKS Ingress / App Service (private, no public IP)
   │
   ▼
Private VNet — banking app tier, no direct internet exposure
```

**Key configuration points for a banking workload:**

1. **WAF in Prevention mode, not Detection** — for a banking app, blocking malicious requests outright is non-negotiable; Detection-only mode just logs and lets the attack through.

2. **OWASP CRS 3.2+ with tuned exclusions** — enable the full core rule set (SQLi, XSS, protocol violations), then tune false-positive exclusions carefully per-endpoint rather than disabling whole rule categories.

3. **Custom rules for banking-specific threats** — rate limiting on login/OTP endpoints (brute-force protection), geo-blocking for regions outside the bank's customer base, and blocking known bad IP reputation lists.

4. **End-to-end TLS, not just edge termination** — TLS terminates at App Gateway for WAF inspection, but re-encrypts to the backend (TLS re-encryption) so traffic inside the VNet is never plaintext — a common compliance requirement (PCI-DSS).

5. **Private backend, no public IP on the app tier** — App Gateway is the only public entry point; AKS/App Service backend sits in a private subnet, reachable only via the Gateway.

6. **Session affinity + connection draining** — for stateful banking sessions, ensure cookie-based affinity is configured so a user's session doesn't bounce between backend instances mid-transaction.

7. **Full logging to Log Analytics/Sentinel** — WAF logs, access logs, and performance logs feed into Azure Sentinel for SIEM correlation — mandatory for banking compliance and fraud detection.

8. **DDoS Protection Standard** on the VNet in addition to WAF — WAF handles L7 attacks, DDoS Protection handles volumetric L3/L4 attacks; banking apps need both layers.

---

## 9. Intermittent DNS resolution issues during Azure deployment

**Common causes, in likely order:**

1. **CoreDNS pod resource limits/throttling on AKS** — under load, CoreDNS pods hit CPU limits and start dropping/delaying resolution. Check:
   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl top pod -n kube-system -l k8s-app=kube-dns
   ```

2. **CoreDNS autoscaling not keeping pace with cluster growth** — default replica count may be too low for the number of nodes/pods doing lookups. Azure recommends the `cluster-proportional-autoscaler` for CoreDNS.

3. **`ndots` configuration causing extra lookups** — Kubernetes' default `ndots:5` in `/etc/resolv.conf` causes every non-fully-qualified DNS query to first try multiple internal search domains before the actual external one, multiplying DNS traffic 4-5x per lookup. This is one of the most common root causes of "intermittent" (really "under load") DNS latency in AKS.

4. **conntrack table race condition** — a well-documented Linux kernel UDP conntrack bug causes DNS queries (UDP) to be dropped under high concurrency due to a race in SNAT/DNAT rule insertion. Symptoms: intermittent 5-second timeouts (matches the UDP retry timeout).

5. **Azure DNS / custom DNS server throttling** — if using a custom DNS forwarder or Azure Firewall as DNS proxy, check for rate limiting or the forwarder's own resource constraints.

6. **VNet peering / private DNS zone propagation delay** during deployment — new resources' DNS records not yet propagated across peered VNets.

**Fix path:** Convert internal service calls to use fully-qualified domain names (`myservice.namespace.svc.cluster.local.` with trailing dot) to avoid the `ndots` search-domain multiplication, scale CoreDNS replicas, and apply the standard mitigation for the conntrack race (`weave works` NodeLocal DNSCache — Azure supports enabling this as an AKS add-on).

---

## 10. 10-second delays every 15 minutes on AKS, no code changes

**Approach — the "every 15 minutes" pattern is the biggest clue; find what else runs on that schedule:**

1. **Check for a CronJob or scheduled reconciliation loop:**
   ```bash
   kubectl get cronjobs -A
   ```
   A backup job, log-rotation job, or metrics-scraping job running every 15 min could be causing resource contention.

2. **Check HPA (Horizontal Pod Autoscaler) scaling events** — HPA polls metrics on an interval (default 15-30s, but scale decisions can have cooldown periods); if pods are scaling up/down on a ~15-min cadence due to a cyclical load pattern, new pods need warm-up time causing latency spikes.

3. **Check node autoscaler activity** — Cluster Autoscaler evaluating and potentially cordoning/draining nodes on its evaluation interval.

4. **Check for a certificate/token refresh cycle** — some sidecars (Istio, cert-manager, Key Vault CSI driver) refresh tokens/certs on a timer; if that refresh briefly blocks the main container's connection pool, it causes periodic latency.

5. **Check DB connection pool recycling** — many ORMs/connection pools recycle idle connections on a timer (very commonly configured at 15 min); if recycling isn't graceful, in-flight requests during that window see delays.

6. **Check GC pause patterns** — for JVM/.NET apps, a full GC cycle triggered by generational thresholds can align with a roughly periodic pattern if load is steady.

7. **Correlate with Azure Monitor/App Insights timeline** — overlay CPU, memory, network, and GC metrics against the exact timestamps of delays to find what spikes in that window.

**Real-world answer pattern:** "No code changes" + fixed interval strongly points to *infrastructure or platform-level scheduled activity* (cron, autoscaler, connection pool recycle, cert rotation) rather than the application logic itself — the RCA process is about correlating timestamps across every scheduled process in the stack, not re-reading app code.

---

## 11. Jenkins jobs randomly failing at artifact upload step

**Layers to check, in order:**

1. **Jenkins agent resource exhaustion** — disk space or memory on the build agent at upload time:
   ```bash
   df -h
   free -m
   ```
   Artifact upload often happens after a large build, so disk fills up right before the step that needs it most.

2. **Network/connectivity to the artifact repository** — transient DNS or TLS handshake failures to Nexus/Artifactory/Azure Artifacts. Check Jenkins console output for timeout vs connection-refused vs SSL errors — each points to a different layer.

3. **Credential/token expiry** — a service account token or API key used for artifact repo auth expiring mid-pipeline-run, especially if it's a short-lived OAuth token not refreshed properly.

4. **Concurrent job contention** — multiple Jenkins jobs uploading to the same artifact path simultaneously, causing a repository-side lock conflict or checksum mismatch.

5. **Repository-side rate limiting or storage quota** — Artifactory/Nexus throttling uploads under load, or the storage backend (e.g., S3/Blob) hitting a quota.

6. **Flaky Jenkins plugin** — outdated Artifactory/Nexus Jenkins plugin versions have known intermittent upload bugs; check plugin changelog against your Jenkins core version for compatibility issues.

7. **Agent-to-controller communication instability** — if using ephemeral/cloud agents (e.g., Kubernetes plugin), the agent pod being evicted or losing connectivity to the Jenkins controller mid-step.

**Debug method:** Enable verbose logging on the specific upload step and correlate failure timestamps against agent resource graphs and network logs — "random" almost always means resource- or network-timing-dependent, not truly random.

---

## 12. Automated rollback strategy in Kubernetes for failed deployments

**Approach:**

1. **Use `kubectl rollout` primitives as the baseline safety net:**
   ```bash
   kubectl rollout status deployment/myapp --timeout=120s
   ```
   If the rollout doesn't reach `Available` status within the timeout, the pipeline treats it as failed and triggers rollback:
   ```bash
   kubectl rollout undo deployment/myapp
   ```

2. **Gate rollout success on readiness probes + real health signals** — don't just check "pods are Running," check that they pass readiness checks and that error rates/latency from Application Insights or Prometheus stay within threshold for a soak period (e.g., 5 minutes) post-deploy.

3. **Automate with a progressive delivery tool** — Argo Rollouts or Flagger integrate directly with Prometheus metrics to automatically analyze canary health and roll back without manual intervention:
   ```yaml
   analysis:
     templates:
     - templateName: success-rate
     args:
     - name: service-name
       value: myapp
   ```
   If the success-rate metric drops below threshold during the canary phase, Argo Rollouts automatically aborts and reverts traffic to the stable version.

4. **CI/CD pipeline-level gate** — in Azure DevOps/GitHub Actions, add a post-deploy verification stage (smoke tests + metric check) that triggers a rollback stage automatically on failure, rather than relying on someone noticing.

5. **Database migration safety** — ensure rollback strategy accounts for schema changes (see Q14) — a pure pod rollback doesn't help if the new version already migrated the DB in a backward-incompatible way.

**Key point for interview:** manual `kubectl rollout undo` is a *reactive* safety net; the senior-level answer is designing *automated, metric-driven* rollback (Argo Rollouts/Flagger) so a bad deploy is caught and reverted within minutes without a human needing to notice first.

---

## 13. Cost-optimized nightly reporting app, 3-year log retention

**Architecture:**

```
Azure Logic App / Function (Timer Trigger, nightly)
        │
        ▼
  Container Instance (ACI) or AKS Job — spins up only for the run duration
        │  — no always-on compute; pay only for actual execution time
        ▼
  Output → Azure Blob Storage (data lake)
        │
        ├── Hot tier (0-30 days)  — recent reports, frequent access
        ├── Cool tier (30 days-1yr) — occasional access
        └── Archive tier (1-3 yrs) — compliance retention, rare access
```

**Key cost decisions:**

1. **Use Azure Container Instances or a Kubernetes CronJob (if AKS already exists) instead of an always-on VM/App Service** — the workload runs once nightly, so paying for 24/7 compute is pure waste. ACI bills per-second only while running.

2. **Blob Storage lifecycle management policy** — automatically transition blobs: Hot → Cool at 30 days → Archive at 1 year, no manual intervention. Archive tier is dramatically cheaper for 3-year compliance retention that's rarely queried.

3. **Avoid Elasticsearch/expensive indexed search for 3-year-old logs** — indexing is expensive; for compliance-only retention (rarely queried), flat files in Archive tier with occasional rehydration-on-demand is far cheaper than keeping everything in a searchable hot store.

4. **Reserved capacity / Azure Savings Plan** only if there's a genuinely predictable baseline — for a nightly batch job, pay-as-you-go on serverless compute is usually cheaper than reservations since there's no steady-state usage to commit to.

5. **Right-size the compute** — nightly reporting jobs are usually I/O-bound (reading logs, generating a report), not CPU-bound; a small SKU is often sufficient — validate with actual run metrics rather than guessing.

6. **Immutable storage policy (WORM)** for the archive tier if compliance requires tamper-proof retention — cheap to add and often a hard requirement for financial/audit logs.

---

## 14. Zero-downtime database migrations

**Approach — the expand/contract pattern:**

1. **Expand phase** — add new schema elements (new column/table) without removing or renaming anything old. Old application code keeps working unmodified.

2. **Dual-write phase** — deploy application code that writes to both old and new schema simultaneously, while still reading from the old schema. This is backward-compatible with any pod still running the previous version during a rolling deploy.

3. **Backfill** — run a background job to migrate/copy existing historical data into the new schema shape, in batches, without locking tables.

4. **Cutover — read from new schema** — once backfill is verified complete and consistent, deploy application code that reads from the new schema (still writing to both, as a safety net).

5. **Contract phase** — after a soak period with no issues, remove the dual-write code and drop the old schema elements in a final cleanup migration.

**Supporting practices:**
- **Use online schema-change tools** for large tables — `gh-ost` or `pt-online-schema-change` for MySQL, or native online DDL for Postgres where available — to avoid table locks during the actual ALTER.
- **Version the API/data contract**, not just the DB — ensures old and new application versions can coexist safely during a rolling Kubernetes deployment.
- **Feature flags** to control which code path (old/new schema) is active per-request, allowing instant rollback without a redeploy if an issue surfaces.
- **Migrations as a separate, idempotent pipeline step** — never bundled directly into the app's startup code where a crash-loop could leave the DB in a half-migrated state.

**Key point for interview:** zero-downtime migration is fundamentally about **never having a single deploy that changes both schema and code assumptions at once** — every step must be backward-compatible with the version running immediately before it.

---

## 15. Disaster recovery for stateful apps running on containers

**Approach:**

1. **Separate stateless (app) from stateful (data) concerns entirely** — the container layer itself should be treated as disposable/re-creatable; DR planning focuses on the data.

2. **Persistent Volume backup strategy** — for AKS, use **Velero** to back up both Kubernetes object state (deployments, configmaps, PVC definitions) and the underlying persistent volume data (via CSI snapshots) on a schedule, replicated to a separate region's storage account.

3. **Database-specific replication, not generic volume snapshots, for actual databases** — e.g., Azure SQL geo-replication, Postgres streaming replication to a standby in another region, or Cosmos DB multi-region writes — application-aware replication is far more reliable than block-level snapshots for consistency guarantees.

4. **Define RPO/RTO explicitly before choosing tooling** — e.g., RPO of 5 minutes requires continuous replication (streaming), while RPO of 24 hours can tolerate nightly snapshot-based backups — this decision drives cost dramatically, so get it agreed with the business first.

5. **Regular restore drills, not just backups** — a backup that's never been test-restored is not a DR plan; schedule quarterly restore-to-a-clean-environment tests and measure actual RTO against the target.

6. **Multi-region AKS with active-passive or active-active** — depending on RTO requirements: active-passive (secondary cluster scaled to zero, Velero-restored on failover) is cheaper; active-active (both regions live, data replicated bidirectionally) gives near-zero RTO but is significantly more complex and costly.

7. **DNS-based failover** — Azure Traffic Manager or Front Door to redirect traffic to the DR region automatically based on health probes, so failover doesn't require manual DNS changes under pressure.

**Key point for interview:** DR for stateful containerized apps is 80% about the **data replication and consistency strategy**, not the container orchestration itself — Kubernetes manifests can be redeployed in minutes; the hard problem is always the database/storage layer.

---

## 16. Azure Function throttling

**Detection:**

1. **Check for `429` responses / HTTP 503 in Application Insights** — throttling shows up as failed requests with specific status codes, or as `FunctionsExceededQuota` in the logs.

2. **Check the hosting plan's scale limits** — Consumption plan has a max instance count (default 200 for HTTP triggers); Premium/Dedicated plans have different ceilings — confirm which plan and what the actual configured/default limit is.

3. **Check for cold-start-induced backlog** — under a traffic spike, Consumption plan functions need to scale out, and cold starts during that scale-out window can cause requests to queue and appear throttled.

4. **Check downstream dependency throttling** — the Function itself might not be throttled by Azure, but a downstream service it calls (Cosmos DB RU limit, SQL DTU limit, another API's rate limit) is returning 429s that bubble up as function failures.

5. **Check `host.json` concurrency settings** — for queue/Event Hub-triggered functions, `maxConcurrentCalls` or `batchSize` misconfigured too low/high can cause artificial throttling or overload.

**Fix:**
- Move to **Premium plan** if Consumption's cold-start/scale ceiling is the bottleneck — Premium keeps warm instances and has higher scale limits.
- Increase downstream capacity (e.g., raise Cosmos DB RU/s, or add retry-with-backoff and a queue buffer in front of the downstream call).
- Tune `host.json` concurrency to match actual downstream capacity rather than maxing it out.
- Add **Application Insights availability alerts** on 429/503 rate specifically so throttling is caught proactively, not reported by users first.

---

## 17. Blue/green deployment with Terraform + rollback

**Plan:**

1. **Provision both environments via Terraform, parameterized by a `color` variable** (blue/green), sharing the same module so both are identical infrastructure:
   ```hcl
   module "app_environment" {
     source = "./modules/app-env"
     color  = var.active_color   # "blue" or "green"
     ...
   }
   ```

2. **Traffic routing via Azure Front Door or Traffic Manager** with a backend pool weight — Terraform manages the weight assignment as a variable, so switching traffic is a `terraform apply` with a changed weight, not a manual portal click:
   ```hcl
   resource "azurerm_frontdoor" "main" {
     backend_pool {
       backend {
         address = module.app_environment["green"].endpoint
         weight  = var.green_weight   # 0 or 100
       }
     }
   }
   ```

3. **Pipeline stages:**
   - Stage 1: Deploy new version to the *inactive* color (e.g., green, while blue serves production).
   - Stage 2: Run smoke tests against green directly (via its own private endpoint, not yet public).
   - Stage 3: Shift Front Door weight to green (100%), keep blue running as instant rollback target.
   - Stage 4: Soak period — monitor error rates/latency for a defined window (e.g., 15 min).
   - Stage 5: If soak passes, decommission/scale down blue (or leave it idle as the next rollback target); if soak fails, **automatically revert the Front Door weight back to blue** — this is the rollback, and it's instant since blue never stopped running.

4. **State management discipline** — use separate Terraform workspaces or state files per color to avoid one `apply` accidentally affecting both environments.

**Key point for interview:** the rollback in blue/green is a **traffic-routing change, not a redeploy** — that's what makes it near-instant (seconds, via DNS/Front Door weight) compared to rolling-update rollback which requires re-pulling and restarting pods.

---

## 18. End-to-end SLA monitoring for a payments pipeline

**Approach:**

1. **Define the SLA in business terms first** — e.g., "99.95% of payment transactions complete in under 2 seconds end-to-end" — this drives what to instrument, rather than starting with tooling.

2. **Distributed tracing across every hop** — instrument the full payment flow (API Gateway → Auth service → Payment service → Fraud check → Bank/PSP integration → Confirmation) with **OpenTelemetry**, propagating a single `trace_id` through every service so you can see the full latency breakdown per transaction, not just per-service averages.

3. **Synthetic transactions** — run a scheduled synthetic "test payment" through the full pipeline every few minutes (Azure Application Insights Availability Tests or a custom synthetic monitor) to catch SLA breaches even during low real-traffic periods.

4. **Real User Monitoring + business-level SLIs** — track actual transaction success rate and P95/P99 latency (not just average) as the primary SLI, since averages hide tail latency that affects real customers.

5. **Dependency-aware alerting** — since payments touch external PSPs/banks outside your control, separately track and alert on external dependency latency vs your own service latency, so on-call can immediately tell if an SLA breach is internal or a third-party outage.

6. **SLA dashboard with error budget tracking** — Grafana/Azure Monitor Workbook showing rolling 30-day SLA compliance and remaining error budget, so the team can make informed release-velocity decisions (freeze risky deploys if the error budget is nearly exhausted).

7. **Alert on SLA burn rate, not just point-in-time breaches** — a multi-window burn-rate alert (e.g., Google SRE-style: fast burn over 1hr AND slow burn over 6hr) catches both sudden outages and slow degradations without excessive noise.

**Key point for interview:** SLA monitoring for a payments pipeline must be **end-to-end and trace-based**, not per-service — a payment can be "successful" in every individual microservice's metrics while still breaching the overall customer-facing SLA due to cumulative latency or a handoff failure between services.

---

## 19. Scaling: compute-intensive vs I/O-intensive

| Aspect | Compute-intensive | I/O-intensive |
|---|---|---|
| **Bottleneck** | CPU cycles (encoding, ML inference, heavy computation) | Waiting on network/disk/DB calls |
| **Scaling metric** | CPU utilization-based HPA | Custom metrics — queue depth, request latency, concurrent connections |
| **VM/node SKU choice** | Compute-optimized SKUs (Fsv2, Fsv2-series) — high CPU-to-memory ratio | General-purpose or memory-optimized (Dsv5, Esv5) — often need more RAM for connection pools/caching, less raw CPU |
| **Scaling pattern** | Vertical scaling helps meaningfully (bigger CPU = faster per-unit work) | Vertical scaling has diminishing returns — the bottleneck is waiting, not processing power; horizontal scaling (more instances handling concurrent I/O) is far more effective |
| **Concurrency model** | Thread/worker pool sized close to core count (more threads than cores just causes context-switch overhead) | Async/non-blocking I/O (event loop model — Node.js, async Python) lets a single instance handle many concurrent I/O-bound requests without needing 1:1 thread-per-request |
| **Autoscaler tuning** | HPA on CPU with fairly standard thresholds (e.g., scale at 70% CPU) | HPA on custom metrics (Prometheus adapter) — e.g., scale based on request queue length or P95 latency, since CPU can look artificially low while requests pile up waiting on I/O |
| **Real example** | Video transcoding service, ML batch inference | API gateway, payment processing service, anything calling external APIs/DBs heavily |

**Key point for interview:** the mistake senior engineers catch juniors making is autoscaling an I/O-bound service purely on CPU% — it'll look "fine" (low CPU) while actually being saturated on concurrent connections or queue depth, so CPU-based HPA alone under-scales it and users see latency long before any CPU alert fires.

---

## 20. Pipeline blocked, stakeholders unreachable

**Approach — balance unblocking work against bypassing governance:**

1. **Assess the actual risk/urgency first** — is this a routine deploy that can genuinely wait, or a critical hotfix (security patch, production-down fix) where delay itself causes harm? The answer path differs significantly.

2. **Check for an escalation path/backup approver** — most mature orgs define a secondary approver or on-call lead specifically for this situation; use it rather than defaulting to "wait" or "bypass."

3. **For non-urgent changes** — simply wait and communicate the delay transparently (e.g., Slack update: "deployment blocked pending approval from X, expected delay of Y") rather than trying to force it through.

4. **For genuinely urgent/critical fixes** — invoke the documented emergency-change process if one exists (most compliance frameworks like SOC2/ISO27001 require one) — this typically allows deployment with retroactive approval logged, under a break-glass procedure, rather than silently skipping the gate.

5. **Never manually bypass approval gates without going through the defined exception process** — even under pressure, an ad-hoc bypass undermines the entire audit trail and compliance posture; if no break-glass process exists, that's a gap to flag and fix afterward, not a reason to improvise one under pressure.

6. **Document everything in real time** — who was contacted, when, via what channel, and what decision was made — this protects both the individual and the organization if questioned later.

7. **Post-incident follow-up** — regardless of outcome, raise it in the next retro: stakeholder unavailability blocking a critical pipeline is a process gap (single point of failure in the approval chain) that should be fixed by adding backup approvers or automating low-risk approval criteria.

**Key point for interview:** the senior-level answer isn't "I'd bypass it to unblock the team" — it's demonstrating you know the **difference between urgent and merely inconvenient**, and that mature orgs have (or should have) a defined emergency-change/break-glass process for exactly this situation, so the right move is invoking that process, not improvising around governance.

---

*Compiled as an interview prep reference for senior DevOps / Cloud / SRE roles focused on Azure and AKS.*
