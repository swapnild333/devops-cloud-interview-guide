# Kubernetes + Linux Networking — Complete Interview Prep (Beginner Friendly)

> **How to use this file:** Every term below is explained in plain English first, then "why it matters in real life," then a CLI example with sample output so you can actually picture it. Technical terms are kept exact (interviewers listen for these words) but the explanation around them is kept simple.

---

# PART 1: The Big Idea (Say This First in Any Networking Answer)

> "Kubernetes does not replace networking. Kubernetes automates and orchestrates normal Linux networking. Underneath every Service, Ingress, and Pod, it's still the Linux kernel doing routing, NAT, firewalling, and packet forwarding — the same stuff that existed before Kubernetes."

If you remember only one sentence for your interview, remember that one. It shows you understand there's no "magic," just automation on top of Linux.

---

# PART 2: Core Linux Networking Concepts (The Foundation)

These are NOT Kubernetes-specific — they are plain Linux concepts that Kubernetes relies on. Interviewers love asking these because they separate "YAML users" from people who understand what's happening underneath.

## 2.1 Network Namespace
**What it is:** A Linux feature that gives a process its own isolated network stack — its own IP address, its own routing table, its own interfaces — completely separate from the host and from other processes.

**Real-life use case:** This is literally how a Pod gets "its own IP." Every Pod = one network namespace. Two containers in the same Pod share the same network namespace, which is why they can talk to each other via `localhost`.

**CLI Example:**
```bash
# List network namespaces on a node
ip netns list

# See a pod's network namespace from inside a node
ls -l /var/run/netns/

# Enter a namespace and check its IP
ip netns exec <namespace-id> ip addr show
```
**Sample output:**
```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
    inet 10.244.2.15/24 scope global eth0
```
This is why `10.244.2.15` is "the pod's IP" — that IP lives inside the pod's own namespace.

---

## 2.2 veth Pair (Virtual Ethernet Pair)
**What it is:** A pair of virtual network cables. Whatever goes in one end comes out the other end. One end sits inside the Pod's network namespace, the other end sits on the host (node).

**Real-life use case:** This is the actual "wire" connecting a Pod to the node's network. Without a veth pair, a pod would be network-isolated with no way to send/receive packets.

**CLI Example:**
```bash
# On the node, list veth interfaces (one per running pod)
ip link show type veth
```
**Sample output:**
```
15: veth3a2f1c@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
```
Each `vethXXXX` you see here is one end of a virtual cable going into a pod.

---

## 2.3 Linux Bridge
**What it is:** A virtual network switch inside the kernel. It connects multiple veth pairs together so pods on the same node can talk to each other.

**Real-life use case:** When Pod A and Pod B are on the same node, their traffic usually goes: Pod A → veth → **bridge** → veth → Pod B, entirely inside the kernel, never leaving the machine.

**CLI Example:**
```bash
# Common bridge name used by many CNI plugins
ip link show cni0
brctl show cni0
```
**Sample output:**
```
bridge name  bridge id          STP enabled  interfaces
cni0         8000.0a58c0a80101  no           veth3a2f1c
                                              veth9b12aa
```

---

## 2.4 iptables
**What it is:** The traditional Linux firewall and packet-rewriting tool. It works by matching packets against rules (chains) and then doing something: ACCEPT, DROP, or REWRITE (NAT).

**Real-life use case:** This is THE most important term in Kubernetes networking interviews. `kube-proxy` uses iptables to make a Kubernetes **Service** (a fake, virtual IP) actually work — it rewrites the destination IP of a packet from the Service's virtual IP to a real Pod's IP.

**CLI Example:**
```bash
# See NAT rules that kube-proxy installed for a Service
iptables -t nat -L KUBE-SERVICES -n | grep backend-api
```
**Sample output:**
```
KUBE-SVC-XYZ123  tcp  --  0.0.0.0/0  10.96.14.23  /* default/backend-api */ tcp dpt:80
```
This line is proof that traffic to `10.96.14.23:80` (the fake Service IP) gets redirected somewhere real.

---

## 2.5 IPVS (IP Virtual Server)
**What it is:** A faster, kernel-level load-balancing alternative to iptables. Same purpose as iptables mode for kube-proxy, but built for performance at scale (hash tables instead of long rule lists).

**Real-life use case:** Clusters with thousands of Services get slow with iptables (rules are checked one by one, top to bottom). IPVS is chosen in large clusters because lookup is O(1) instead of O(n).

**CLI Example:**
```bash
# Check which mode kube-proxy is running in
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# View IPVS virtual server table
ipvsadm -L -n
```
**Sample output:**
```
mode: "ipvs"

TCP  10.96.14.23:80 rr
  -> 10.244.2.15:8080   Masq  1  0  0
  -> 10.244.3.9:8080    Masq  1  0  0
```
`rr` = round robin. You can literally see the load balancing algorithm here.

---

## 2.6 NAT (Network Address Translation)
**What it is:** Rewriting the source or destination IP/port of a packet as it passes through.

**Real-life use case:** Two kinds matter in Kubernetes:
- **DNAT** (Destination NAT) — rewrites Service IP → real Pod IP (this is what kube-proxy does).
- **SNAT** (Source NAT) — rewrites a Pod's internal IP → the Node's IP when talking to something outside the cluster (like the internet), so the return traffic knows how to come back.

**Real-life analogy:** Like a receptionist who takes a call meant for "Extension 100" and forwards it to the actual desk phone that's ringing — the caller never needs to know the real extension.

**CLI Example:**
```bash
iptables -t nat -L POSTROUTING -n | grep MASQUERADE
```
**Sample output:**
```
MASQUERADE  all  --  10.244.0.0/16  0.0.0.0/0
```
This means: "Any pod traffic (10.244.0.0/16) leaving the cluster gets masqueraded (SNAT'd) behind the node's IP."

---

## 2.7 conntrack (Connection Tracking)
**What it is:** The Linux kernel's memory table of every active connection — source IP/port, destination IP/port, protocol, state. NAT literally cannot work without conntrack because the kernel needs to remember "I rewrote this packet, so I must rewrite the reply too."

**Real-life use case:** This is a classic **production incident term**. If the conntrack table fills up (too many connections), new connections start getting dropped — and it looks like "random Kubernetes networking is broken" when it's really a Linux kernel limit.

**CLI Example:**
```bash
# Current number of tracked connections vs max allowed
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```
**Sample output:**
```
262143
262144
```
That's basically 100% full — the next new connection will likely get dropped. In logs you'd see:
```
nf_conntrack: table full, dropping packet
```
**Interview gold line:** "When teams say 'Kubernetes networking randomly breaks under load,' 9 times out of 10 I'd check conntrack exhaustion before blaming Kubernetes."

---

## 2.8 MTU (Maximum Transmission Unit)
**What it is:** The largest packet size (in bytes) that can be sent over a network link without being broken into pieces (fragmented).

**Real-life use case:** Overlay networks (see VXLAN below) wrap your packet inside another packet, adding extra bytes as "headers." If the node's MTU is 1500 but the overlay adds 50 bytes of overhead, the pod's *effective* usable MTU is only 1450. If an app doesn't know this and sends 1500-byte packets, they get fragmented or silently dropped — causing weird, intermittent failures like broken TLS handshakes or random gRPC errors.

**CLI Example:**
```bash
# Check node interface MTU
ip link show eth0 | grep mtu

# Check pod interface MTU (should be lower if using overlay)
kubectl exec -it mypod -- ip link show eth0 | grep mtu

# Test with a large ping to catch fragmentation issues
ping -M do -s 1472 8.8.8.8
```
**Sample output:**
```
eth0: mtu 1500          (node)
eth0: mtu 1450          (inside pod, due to VXLAN overhead)
```
**Real symptom:** Small HTTP requests work fine, but large gRPC payloads or file uploads randomly fail — classic MTU mismatch symptom.

---

## 2.9 ARP (Address Resolution Protocol)
**What it is:** How a machine finds the MAC (hardware) address that belongs to a given IP address on the same local network segment.

**Real-life use case:** Before Node A can actually send a packet to Node B on the same local subnet, it needs the destination's MAC address — ARP is the "who has this IP, tell me your MAC" broadcast that makes that possible.

**CLI Example:**
```bash
ip neigh show
```
**Sample output:**
```
10.244.2.1 dev eth0 lladdr 0a:58:0a:f4:02:01 REACHABLE
```

---

## 2.10 cgroups (Control Groups)
**What it is:** A Linux kernel feature that limits and tracks how much CPU, memory, and I/O a process (or group of processes) can use.

**Real-life use case:** This is literally what makes Kubernetes **resource requests/limits** real. When you write `resources.limits.memory: 512Mi` in a Pod spec, Kubernetes translates that into a cgroup rule enforced by the kernel — if the container tries to exceed it, the kernel kills it (OOMKilled).

**CLI Example:**
```bash
# Find the cgroup for a container and check its memory limit
cat /sys/fs/cgroup/memory/kubepods/.../memory.limit_in_bytes
```
**Sample output:**
```
536870912   # exactly 512Mi in bytes
```
**Interview gold line:** "Resource limits aren't a Kubernetes invention — they're a YAML-friendly wrapper around Linux cgroups."

---

## 2.11 Linux Namespaces (General Concept)
**What it is:** cgroups control *how much* a process can use; namespaces control *what a process can see*. There are several kinds: network, PID, mount, UTS (hostname), IPC, user.

**Real-life use case:** This is the actual technology that makes a "container" a container. A Pod's isolation — its own hostname, its own process list, its own network — comes from namespaces, not from any special Kubernetes magic.

**CLI Example:**
```bash
lsns -t net
```
**Sample output:**
```
NS TYPE NPROCS PID USER   COMMAND
4026532567 net  2   50123 root   /pause
```

---

## 2.12 XDP (eXpress Data Path)
**What it is:** A way to run custom packet-processing code at the earliest possible point — right when the packet arrives at the network card driver — before it even enters the normal Linux networking stack.

**Real-life use case:** Used for extremely fast packet filtering/dropping, e.g., DDoS mitigation, or by eBPF-based CNIs like Cilium to make forwarding decisions with almost no overhead.

---

## 2.13 eBPF (extended Berkeley Packet Filter)
**What it is:** A technology that lets you run small, safe, sandboxed programs *inside the Linux kernel itself*, without writing kernel modules or adding extra userspace processes (like sidecars).

**Real-life use case:** Instead of routing every packet through iptables rule-by-rule, or adding an Envoy sidecar container to every pod for observability, eBPF programs can watch/redirect/filter traffic directly in the kernel. This is why **Cilium** (an eBPF-based CNI) is popular — faster networking, faster observability, no sidecar overhead.

**CLI Example:**
```bash
# List loaded eBPF programs on a node running Cilium
bpftool prog list
```
**Sample output:**
```
142: sched_cls  name cil_from_container  tag a1b2c3
```
**Interview gold line:** "eBPF replaces a chain of iptables rules and sidecar proxies with a single program running inside the kernel — that's why it's faster and lighter."

---

## 2.14 TCP Socket
**What it is:** The actual endpoint of a network connection inside an application — identified by an IP + port + protocol.

**Real-life use case:** When you say "the app has a connection pool," you're really talking about a pool of open TCP sockets. Exhausting available sockets/ports is a real production failure mode (e.g., "SNAT port exhaustion" on cloud load balancers).

**CLI Example:**
```bash
# See open connections from inside a pod/node
ss -tnp
```
**Sample output:**
```
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
ESTAB   0       0        10.244.2.15:41022    10.96.14.23:80
```

---

# PART 3: Kubernetes Networking Layers (The Stack)

Kubernetes networking is not ONE system — it's several systems stacked on top of each other. Knowing this list, and being able to say which layer a problem belongs to, is exactly what separates senior candidates.

| Layer | Responsibility | Real Technology Underneath |
|---|---|---|
| Pod networking | Pod-to-pod communication | veth, bridge, CNI plugin |
| Service networking | Stable virtual IP + load balancing | iptables / IPVS, kube-proxy |
| DNS | Internal service discovery | CoreDNS |
| Ingress/Gateway | External traffic routing in | Ingress controller (nginx, etc.) |
| Overlay network | Cross-node communication | VXLAN, BGP, eBPF routing |
| Cloud networking | VPC/subnet/load balancer integration | ENIs, SNAT, NAT gateways |
| Service mesh | mTLS, retries, observability | Envoy sidecars (Istio, Linkerd) |
| Network policies | Firewall-like control | iptables/eBPF rules from NetworkPolicy objects |
| eBPF systems | Kernel-level packet handling | Cilium, XDP |
| Observability | Metrics/traces/log transport | Prometheus, OpenTelemetry, eBPF probes |

**Interview tip:** If asked "how does Kubernetes networking work," don't just describe pods — walk through this table top to bottom. It shows structured thinking.

---

# PART 4: Full Walkthrough — What REALLY Happens on One `curl` Call

This is the single best answer you can give in an interview because it proves end-to-end understanding.

**Scenario:**
```
frontend pod  →  curl http://backend-api  →  backend pod  →  postgres
```

**The Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

### Step 1 — DNS Resolution
The pod's `/etc/resolv.conf` points to CoreDNS. It asks:
```
backend-api.default.svc.cluster.local
```
CoreDNS answers with the Service's virtual IP (**ClusterIP**):
```
10.96.14.23
```
**Systems involved:** CoreDNS, kubelet's DNS config injection, the pod's `/etc/resolv.conf`.

```bash
kubectl exec -it frontend -- nslookup backend-api
```
```
Name:   backend-api.default.svc.cluster.local
Address: 10.96.14.23
```

### Step 2 — kube-proxy Rewrites the Packet (DNAT)
`10.96.14.23` is **not a real interface** — nothing actually "owns" that IP. It only exists as an iptables/IPVS rule.
```
10.96.14.23:80  →  rewritten to  →  10.244.2.15:8080
```
This is pure Linux **NAT** — one packet rewrite, done entirely in the kernel, invisible to the application.

### Step 3 — Overlay Network (If Pods Are on Different Nodes)
If `10.244.2.15` (the backend pod) lives on a different node than the frontend pod, the **CNI plugin** must carry the packet across nodes:
- **Flannel** → wraps the packet using **VXLAN** encapsulation
- **Calico** → uses **BGP** routing (no encapsulation, real routing)
- **Cilium** → uses **eBPF** routing
- **Cloud-native CNIs** (like AWS VPC CNI) → uses real VPC routing, pod IP is a real routable IP

### Step 4 — conntrack Tracks the Flow
The kernel records this connection (source IP/port, dest IP/port, state) so return traffic can be un-rewritten correctly. If this table fills up cluster-wide → random timeouts, DNS failures, latency spikes — often wrongly blamed on "Kubernetes."

### Step 5 — Response Comes Back
The reply packet arrives, conntrack recognizes the flow, reverses the NAT rewrite, and the frontend pod sees a normal HTTP response — with zero awareness that any of steps 1–4 happened.

---

# PART 5: CNI Plugins Compared

**What CNI (Container Network Interface) is:** A standard/plugin system that Kubernetes calls to actually set up a pod's networking (assign IP, create veth, attach to bridge, set routes) whenever a pod is created.

| CNI | How it moves packets across nodes | Notes |
|---|---|---|
| **Flannel** | VXLAN encapsulation (wraps packet in a packet) | Simple, but overhead + MTU issues |
| **Calico** | BGP routing (no wrapping, real routing) | Fast, but needs network team buy-in for BGP |
| **Cilium** | eBPF (kernel-level program routes packets) | Fast, rich observability, replaces need for some sidecars |
| **Cloud CNIs (AWS VPC CNI, etc.)** | Native cloud VPC routing | Pod IP = real routable cloud IP, but limited by ENI capacity |

```bash
kubectl get pods -n kube-system -o wide | grep -Ei 'flannel|calico|cilium'
```

---

# PART 6: Cloud Networking Gotchas (Not "Kubernetes Bugs" — Cloud Limits)

## 6.1 ENI Exhaustion (AWS-specific but concept applies elsewhere)
**What it is:** In AWS EKS, pods get real IPs from Elastic Network Interfaces (ENIs) attached to the node's EC2 instance. Each instance type has a hard limit on how many ENIs/IPs it can hold.

**Real-life symptom:**
```bash
kubectl describe pod mypod
```
```
Warning  FailedScheduling  ... Insufficient pods
Warning  FailedCreatePodSandBox ... failed to assign an IP address to container
```
Node still has free CPU/memory — but it's out of usable IPs. This is a **cloud networking limit**, not a Kubernetes bug.

| Instance Type | Approx Max Pods (IP-limited) |
|---|---|
| t3.medium | ~17 |
| m5.large | ~29 |
| c5.2xlarge | ~58 |

## 6.2 SNAT Port Exhaustion
**What it is:** When many pods on a node talk to the internet, they all get SNAT'd behind the same node/NAT gateway IP. Each NAT mapping needs a unique source port — there are only ~64,000 ports available. Under heavy outbound traffic, you run out.

**Symptom:** Random outbound connection failures under load, especially to external APIs — even though the app and cluster both "look healthy."

---

# PART 7: Service Mesh Layer (Istio / Linkerd)

**What a service mesh is:** A layer that injects a proxy (**sidecar**, usually **Envoy**) into every pod, so it can add mTLS encryption, retries, traffic policy, and observability — without changing application code.

**Real request path becomes:**
```
frontend app
   ↓
Envoy sidecar (frontend)
   ↓ mTLS encrypted
Envoy sidecar (backend)
   ↓ decrypt, apply policy, retries
backend app
```

**Real-life use case:** Zero-trust security (every internal call is encrypted automatically), automatic retries on transient failures, and detailed per-request metrics — without every team writing that logic into every app.

**Why it makes debugging harder:** A latency spike could now be caused by:
- Envoy CPU saturation
- mTLS certificate rotation delays
- Retry storms (a failing service gets hit even harder because everyone auto-retries)
- Sidecar memory pressure
- Connection pool exhaustion at the proxy layer

```bash
# Check sidecar resource usage — often the real bottleneck
kubectl top pod frontend --containers
```
```
POD        NAME              CPU(cores)   MEMORY
frontend   frontend-app      50m          80Mi
frontend   istio-proxy       310m         210Mi   <-- sidecar using more than the app!
```

---

# PART 8: Quick-Fire Definitions Table (Last-Minute Review)

| Term | One-Line Definition | Where You'll See It Fail |
|---|---|---|
| **Pod IP** | Unique IP per pod, from CNI | Pod can't reach another pod → check CNI |
| **ClusterIP** | Virtual stable IP for a Service | curl to Service hangs → check kube-proxy rules |
| **kube-proxy** | Programs iptables/IPVS to route Service traffic to pods | Service exists but no traffic reaches pods |
| **CoreDNS** | Cluster's internal DNS server | `nslookup` fails inside pod, but ping to IP works |
| **CNI plugin** | Sets up pod networking (IP, veth, routes) | Pod stuck in `ContainerCreating` |
| **Ingress Controller** | Reverse proxy for external → internal HTTP traffic | 404/502 from outside, but Service works internally |
| **NetworkPolicy** | Kubernetes-native firewall rules between pods | Pod can't reach another pod despite Service working |
| **iptables** | Kernel firewall/NAT rule engine | Look here first for any Service routing issue |
| **conntrack** | Tracks active connections for NAT | "table full" errors under high load |
| **MTU mismatch** | Overlay adds header bytes, shrinking usable packet size | Large payloads fail, small ones work fine |
| **VXLAN** | Encapsulates packets to cross nodes (used by Flannel) | Adds MTU overhead, adds slight latency |
| **BGP** | Real routing protocol (used by Calico, no encapsulation) | Needs network team coordination |
| **eBPF** | Kernel-level programmable packet handling | Used by Cilium; replaces some iptables/sidecar overhead |
| **cgroups** | Enforces CPU/memory limits at kernel level | OOMKilled pods, CPU throttling |
| **Network Namespace** | Gives a pod its own isolated network stack | Foundation of "every pod has its own IP" |
| **veth pair** | Virtual cable connecting pod namespace to node | Broken pod networking at the lowest level |
| **Linux bridge** | Virtual switch connecting pods on same node | Same-node pod-to-pod traffic issue |
| **SNAT** | Rewrites source IP (pod → node IP) for outbound traffic | Outbound internet calls failing under load |
| **DNAT** | Rewrites destination IP (Service IP → pod IP) | This is literally how kube-proxy routing works |
| **ENI exhaustion** | Cloud limit on IPs per node (AWS) | "Insufficient pods" errors with free CPU/memory |
| **Sidecar (Envoy)** | Per-pod proxy container for mesh features | Extra latency/CPU/memory per request |
| **mTLS** | Mutual TLS — both sides of a connection authenticate | Cert rotation issues cause mesh connection failures |

---

# PART 9: How to Answer "How Does Kubernetes Networking Work?" in an Interview

Use this structure (talk for ~90 seconds):

1. **One-liner:** "Kubernetes automates Linux networking — it doesn't replace it. Underneath, it's still iptables/IPVS, network namespaces, and the kernel."
2. **Pod-to-pod:** "Every pod gets its own network namespace and IP via the CNI plugin, connected to the node through a veth pair and a Linux bridge."
3. **Service networking:** "A Service is a virtual IP. kube-proxy programs iptables or IPVS rules so that traffic to that virtual IP gets DNAT'd to a real pod IP."
4. **Cross-node:** "If pods are on different nodes, the CNI plugin handles that — VXLAN encapsulation for Flannel, BGP for Calico, or eBPF for Cilium."
5. **DNS:** "CoreDNS resolves internal service names to ClusterIPs."
6. **Ingress:** "External traffic comes in through an Ingress controller, which is really just a reverse proxy Kubernetes manages for you."
7. **Close strong:** "And when things break, it's usually not 'Kubernetes broken' — it's conntrack exhaustion, MTU mismatches, or a cloud-level IP limit like ENI exhaustion, showing up through Kubernetes."

That last line is the one that makes senior interviewers sit up.
