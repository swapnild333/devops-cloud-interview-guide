# 🌐 Networking Issues

Networking problems are the hardest to debug because the failure can be in DNS, the Service, the NetworkPolicy, kube-proxy, or the CNI plugin — and they all look the same from the outside ("can't connect").

## The Networking Debug Toolkit

```bash
# Your best debugging friend - a container with networking tools
# Deploy a debug pod with full network tools
kubectl run netshoot --rm -it \
    --image=nicolaka/netshoot \
    --restart=Never \
    -n production
# nicolaka/netshoot has: curl, dig, nslookup, netcat, tcpdump, traceroute, ss

# Inside netshoot:
# Test DNS resolution
dig myapp-service.production.svc.cluster.local

# Test service connectivity
curl -v http://myapp-service.production.svc.cluster.local:80/health

# Test if pod IP is reachable directly (bypass Service)
curl -v http://10.0.1.45:3000/health
# Check open connections

ss -tuln
# Trace the route
traceroute myapp-service
```

## DNS Issues

DNS is responsible for ~40% of Kubernetes networking bugs in my experience.

```bash
# Check if CoreDNS is healthy
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                   READY   STATUS    RESTARTS
# coredns-xxx-aaa        1/1     Running   0          ← Both must be Running
# coredns-xxx-bbb        1/1     Running   0

# Check CoreDNS logs

kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# Debug DNS from inside the cluster

kubectl run dns-debug --rm -it --image=busybox --restart=Never -- sh
# Inside:
# nslookup kubernetes.default        ← Test cluster DNS baseline
# nslookup myapp-service.production  ← Test service discovery
# nslookup google.com                ← Test external DNS
# Full DNS name format in Kubernetes:

# <service-name>.<namespace>.svc.<cluster-domain>
# myapp-service.production.svc.cluster.local
# If DNS works from one namespace but not another:
kubectl exec -it myapp-xxx -n production -- cat /etc/resolv.conf
# nameserver 10.96.0.10         ← Should point to CoreDNS ClusterIP
# search production.svc.cluster.local svc.cluster.local cluster.local

# options ndots:5
# ndots:5 means queries with < 5 dots try search domains first
# This can cause slow DNS for external names - each search domain gets tried
# Fix for external DNS performance:
# options ndots:2  (in pod's /etc/resolv.conf or via dnsConfig)
```

```yaml
# Configure DNS per-pod for performance
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"     # Reduces DNS lookup chains for external names
      - name: timeout
        value: "1"
  dnsPolicy: ClusterFirst
```

## Service Not Routing Traffic

```bash
# Step 1: Verify the Service exists and has endpoints
kubectl get svc myapp-service -n production
# NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# myapp-service   ClusterIP   10.96.45.123   <none>        80/TCP    5d

# Step 2: Check if Service has endpoints (CRITICAL - no endpoints = selector mismatch)
kubectl get endpoints myapp-service -n production
# NAME            ENDPOINTS                               AGE
# myapp-service   <none>    ← No endpoints! Selector not matching any pods.
# OR:

# NAME            ENDPOINTS                               AGE
# myapp-service   10.0.1.45:3000,10.0.2.67:3000          ← Good - pods found

# Step 3: If no endpoints - check selector match
kubectl get svc myapp-service -n production -o jsonpath='{.spec.selector}'
# {"app":"myapp","version":"v2"}   ← Service selector
kubectl get pods -n production -l app=myapp,version=v2
# No pods found!   ← Pods have different labels - selector mismatch!

# Check what labels pods actually have
kubectl get pods -n production --show-labels
# NAME       READY  STATUS   LABELS
# myapp-xxx  1/1    Running  app=myapp,version=v2.0.0   ← version tag differs!

# Fix: Update service selector to match actual pod labels
kubectl patch svc myapp-service -n production \
    -p '{"spec":{"selector":{"app":"myapp"}}}'   # Remove the version selector
```

## Network Policy Blocking Traffic

```bash
# Check if NetworkPolicies exist in the namespace
kubectl get networkpolicies -n production

# Test if a policy is blocking you - try from both directions

# From inside the cluster, test the connection that's failing:
kubectl exec -it myapp-xxx -n production -- \
    curl -v --max-time 5 http://postgres-service:5432
# If it times out (not refused) → likely NetworkPolicy blocking

# If it's refused immediately → connection reached but nothing listening
# Describe the NetworkPolicy to understand the rules
kubectl describe networkpolicy deny-all -n production

# Temporarily label a pod to test policy inclusion
kubectl label pod myapp-xxx -n production test=allowed

# Check egress from the app's perspective
kubectl exec -it myapp-xxx -n production -- \
    curl -v --max-time 5 http://external-api.example.com
# Timeout? Check if egress to external IPs is blocked by a NetworkPolicy
```

## kube-proxy / iptables Issues

```bash
# Check kube-proxy is running on the affected node
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
# Find which node the failing pod is on, confirm kube-proxy is Running there

# Check kube-proxy logs on the problematic node
NODE=$(kubectl get pod myapp-xxx -n production -o jsonpath='{.spec.nodeName}')
PROXY_POD=$(kubectl get pods -n kube-system -l k8s-app=kube-proxy \
    --field-selector spec.nodeName=${NODE} -o name)
kubectl logs ${PROXY_POD} -n kube-system --tail=50

# Verify iptables rules on the node (requires node access)
# Use kubectl debug to get node shell
kubectl debug node/${NODE} -it --image=ubuntu -- bash

# Inside:
# iptables -t nat -L KUBE-SERVICES | grep <service-clusterip>
# Should see rules for your service
```

---

## Quick Recap

| Layer | Symptom | Key command | Likely cause |
|-------|---------|--------------|----------------|
| DNS | `nslookup` fails or is slow | `dig`, `nslookup`, check `/etc/resolv.conf` | CoreDNS down, wrong `ndots`, search domain chain |
| Service | Connection refused/no route | `kubectl get endpoints` | Selector/label mismatch |
| NetworkPolicy | Connection *times out* (not refused) | `curl --max-time 5`, `describe networkpolicy` | Policy blocking ingress/egress |
| kube-proxy | Service works on some nodes, not others | `kubectl logs` on kube-proxy pod, `iptables -t nat -L` | kube-proxy not running / stale iptables rules |

**Diagnostic signal to remember:** *timeout* usually means something (a policy/firewall) is silently dropping packets; *connection refused* means the packet arrived but nothing is listening on that port.

**Tool of choice:** `nicolaka/netshoot` — has `curl`, `dig`, `nslookup`, `netcat`, `tcpdump`, `traceroute`, `ss` all in one debug image.

**Previous:** [← Init Container Failures](./06-init-container-failures.md) | **Next:** [Node-Level Issues →](./08-node-level-issues.md)
