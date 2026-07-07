# Networking Commands — Senior DevOps Interview Guide

A quick-reference cheat sheet covering the core Linux networking commands, real interview Q&A, and full incident-style scenarios used to evaluate senior DevOps/SRE candidates.

---

## Table of Contents

- [Command Reference](#command-reference)
- [Q&A by Command](#qa-by-command)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Command Reference

| # | Command | Purpose | Example |
|---|---------|---------|---------|
| 1 | `ping` | Check if a host is reachable | `ping google.com` |
| 2 | `ip addr` / `ifconfig` | Show IP address and network interfaces | `ip addr show` |
| 3 | `ip route` | Show routing table | `ip route` |
| 4 | `traceroute` / `tracepath` | Find the path packets take to a destination | `traceroute google.com` |
| 5 | `nslookup` / `dig` | Troubleshoot DNS resolution | `dig google.com` |
| 6 | `curl` | Test APIs and web services | `curl -v google.com` |
| 7 | `wget` | Download files from a URL | `wget example.com/file.zip` |
| 8 | `netstat` / `ss` | Show listening ports and active connections | `ss -tulnp` |
| 9 | `telnet` / `nc` (netcat) | Test connectivity to a port | `nc -zv google.com 80` |
| 10 | `arp` / `ip neigh` | View ARP and neighbor table | `ip neigh` |
| 11 | `hostname` | Show or set system hostname | `hostname` |
| 12 | `iptables` / `nftables` | Manage firewall rules | `iptables -L -n -v` |

---

## Q&A by Command

### 1. ping

**Q: A pod can't reach an external API. First command you run?**
A: `ping api.example.com` to check basic reachability. Many corporate firewalls block ICMP, so a failed ping doesn't confirm the service is down — follow up with `curl -v` to test the actual port/protocol before concluding anything.

**Q: Ping works but the application still times out. Why?**
A: ICMP and the application's actual port (e.g., 443) are handled by different rules in firewalls/security groups. Ping only proves L3 reachability, not that the specific service port is open.

### 2. ip addr / ifconfig

**Q: A container isn't getting network traffic. How do you check its IP configuration?**
A: `ip addr show` inside the container/pod to confirm it has an IP in the expected subnet/CIDR range and the interface is UP. If it's missing an IP, suspect a CNI plugin failure or DHCP issue.

**Q: Why is `ip addr` preferred over `ifconfig` now?**
A: `ifconfig` is deprecated/removed on many modern distros (part of unmaintained net-tools); `ip` (iproute2) is the current standard and supports more features like VRFs and advanced routing.

### 3. ip route

**Q: Two EC2 instances in different subnets within the same VPC can't talk to each other. What do you check?**
A: `ip route` on both hosts to confirm a route exists to the peer subnet, then cross-check the actual VPC route table/NACLs on the cloud side.

**Q: What does a missing default route look like, and what breaks?**
A: No `default via <gateway>` entry — the host can talk to its local subnet but nothing outside it, including DNS resolution to external resolvers.

### 4. traceroute / tracepath

**Q: Latency spikes to a service only in one region. How do you isolate where the delay is introduced?**
A: `traceroute` (or `mtr` for continuous stats) to see per-hop latency and identify which hop introduces delay — points to an ISP, cloud peering point, or misconfigured routing rather than the app itself.

**Q: traceroute shows `* * *` at some hops but still reaches the destination — is that a problem?**
A: Not necessarily — some routers deprioritize or drop ICMP/UDP probes used by traceroute (rate limiting) but still forward actual traffic fine.

### 5. nslookup / dig

**Q: New DNS record isn't resolving after a Route53/CloudFlare change. How do you debug?**
A: `dig example.com` to see the actual answer, TTL, and which nameserver responded (`dig +trace` for the full resolution path). Also check `dig @8.8.8.8` vs the local resolver to isolate propagation delay vs. local DNS cache.

**Q: dig vs nslookup — why do senior engineers usually prefer dig?**
A: `dig` gives more detail (TTL, authority section, flags) and scriptable output, better for automation and deeper debugging; `nslookup` is more interactive/legacy.

### 6. curl

**Q: An API returns 200 in Postman but fails from a server via cron job. What do you check?**
A: `curl -v` from that server to see the full request/response, TLS handshake, headers, and redirects — often reveals missing auth headers, DNS resolution differences, or an expired cert not trusted by the server's CA bundle.

**Q: How do you test a specific TLS version or check certificate expiry with curl?**
A: `curl -v --tlsv1.2 https://example.com` and inspect certificate info in the verbose output, or `curl -vI` for headers-only with handshake details.

### 7. wget

**Q: Why would you use wget instead of curl in a deployment script?**
A: `wget` is better for straightforward recursive downloads and resuming interrupted transfers (`wget -c`) — useful when pulling large artifacts/binaries in CI pipelines where the connection might drop.

**Q: A wget download in a CI job hangs indefinitely — what's your first flag to add?**
A: `--timeout` and `--tries` to fail fast instead of hanging the pipeline; also `-v` to see where it's stuck (DNS, connect, or transfer).

### 8. netstat / ss

**Q: A service claims to be running but clients get "connection refused." How do you verify it's actually listening?**
A: `ss -tulnp` (or legacy `netstat -tulnp`) to confirm the process is bound to the expected port and interface — 0.0.0.0 vs 127.0.0.1 is a classic gotcha.

**Q: Why is `ss` recommended over `netstat` today?**
A: `ss` is faster (reads directly from kernel netlink instead of /proc), and `netstat` is deprecated on modern Linux distros.

### 9. telnet / nc (netcat)

**Q: You suspect a security group/firewall is blocking a specific port, not full connectivity. How do you confirm?**
A: `nc -zv host 443` (or `telnet host 443`) to test just that TCP port without needing the full application protocol.

**Q: How would you quickly test if a load balancer health check port is reachable from a bastion host?**
A: `nc -zv <lb-ip> <health-check-port>` — if it fails, check security groups/NACLs before assuming the app is unhealthy.

### 10. arp / ip neigh

**Q: Two hosts on the same L2 subnet can't reach each other despite correct IP config. What do you check?**
A: `ip neigh` (or `arp -a`) to see if MAC address resolution is happening — stale/incomplete ARP entries can indicate a duplicate IP conflict or VLAN misconfiguration.

**Q: When would you manually clear the ARP cache?**
A: After a failover event (e.g., VIP moved to a different host) where clients still have the old MAC cached, sending traffic to the wrong (now-standby) node.

### 11. hostname

**Q: Why would you check `hostname` during a Kubernetes node troubleshooting session?**
A: To confirm the node's hostname matches what's registered in the cluster (`kubectl get nodes`) — mismatches can cause certificate/kubelet registration issues, especially after cloning VM images.

**Q: What's a common hostname-related issue in cloud auto-scaling groups?**
A: Duplicate hostnames from cloned AMIs/images causing conflicts in service discovery or cluster join failures — cloud-init should regenerate a unique hostname per instance.

### 12. iptables / nftables

**Q: A pod-to-pod connection works on one node but not another in the same Kubernetes cluster. What do you inspect?**
A: `iptables -L -n -v` (or `nft list ruleset`) to check for CNI-managed rules, look for DROP counters incrementing on the affected chain, and compare rule sets between the healthy and unhealthy node.

**Q: Why are companies moving from iptables to nftables?**
A: nftables offers better performance at scale (single unified framework), atomic rule updates, and more efficient rule matching — relevant for large K8s clusters using heavy NAT/firewall rules.

---

## Scenario-Based Questions

### Scenario 1: Website down, monitoring shows server healthy

1. `ping example.com` — check if the domain resolves and responds
2. `dig example.com` — verify DNS returns the correct IP (catches stale DNS, wrong record, or hijack)
3. `curl -v https://example.com` — see the actual HTTP response, TLS handshake, and redirect chain
4. `ss -tulnp` on the server — confirm the app is actually listening on 443, not just running

**What they're testing:** Do you understand "server healthy" (CPU/memory) ≠ "service reachable"? Usually lands on DNS pointing to an old IP after failover, or a security group change blocking 443.

### Scenario 2: New pod on Node B can't reach a DB that Node A pods reach fine

1. `ip addr` inside the pod — confirm it got a valid IP from the CNI
2. `ip route` — check for a route to the DB subnet
3. `ping <db-ip>` from the pod — basic reachability
4. `nc -zv <db-ip> 5432` — confirm the actual DB port (ICMP might be blocked separately)
5. `iptables -L -n -v` on Node B — compare against Node A, look for DROP counters

**What they're testing:** Node-specific iptables/CNI rule drift — a real incident pattern from manual rules or kube-proxy sync failures.

### Scenario 3: SSH works but the container inside can't reach the internet

1. `ip addr` inside container — confirm interface is up
2. `ip route` inside container — check default gateway exists
3. `ping 8.8.8.8` — test raw IP reachability (rules out DNS)
4. `dig google.com` / `nslookup google.com` — if ping-by-IP works but this fails, it's DNS resolution inside the container (broken `/etc/resolv.conf` or Docker DNS config)
5. `curl -v https://google.com` — confirm if it's a proxy/TLS issue on top of that

**What they're testing:** Layered diagnosis — separating L3 connectivity from DNS from application-layer issues instead of guessing.

### Scenario 4: Intermittent 502s from the load balancer, only during peak traffic

1. `ss -s` or `ss -tulnp` — check for exhausted ephemeral ports or excessive TIME_WAIT connections
2. `netstat -an | grep TIME_WAIT | wc -l` — quantify it
3. `dig` — rule out DNS-based load balancing sending traffic to a dead backend
4. `curl -v` repeatedly against individual backend IPs directly (bypassing LB) — isolate a specific backend node
5. `iptables -L -n -v` — check for conntrack table exhaustion (`conntrack -L | wc -l` as a follow-up)

**What they're testing:** Knowledge that port exhaustion and conntrack table limits are common root causes under load, not just "the app is slow."

### Scenario 5: A newly launched EC2 instance from a golden AMI fails to join the cluster

1. `hostname` — check for a duplicate/stale hostname baked into the AMI
2. `ip addr` — confirm it got a fresh IP, not a cached one
3. `ip route` — verify default gateway is correct for its subnet
4. `arp -a` / `ip neigh` — check for IP/MAC conflicts during scaling
5. `dig` — confirm resolution of internal service discovery names (etcd, cluster API)

**What they're testing:** Understanding that AMI cloning without proper cloud-init/hostname regeneration causes real cluster join failures.

### Scenario 6: Traceroute shows packet loss at a specific hop — is the network down?

**Expected pushback:** Not necessarily. `traceroute`/`tracepath` uses ICMP or UDP probes that many routers deprioritize or rate-limit for control-plane protection, even while forwarding real traffic fine.

1. `mtr` for sustained loss stats over time (not a one-off snapshot)
2. `curl -w "%{time_total}"` to measure real application-level latency/loss
3. Only escalate if real traffic (curl/nc) shows actual degradation, not just traceroute hops

**What they're testing:** Whether you blindly trust traceroute or understand its limitations.

### Scenario 7: A firewall change last night broke overnight deployments pulling artifacts

1. `nc -zv artifactory.internal 443` — test if the port is reachable, isolating from DNS/app issues
2. `curl -v https://artifactory.internal` — see if it's a TLS/cert issue vs a connection-level block
3. `ip route` — confirm no routing change broke the path
4. `iptables -L -n -v` / `nft list ruleset` — check counters on relevant chains, diff against the pre-change ruleset (usually where the answer is)
5. `traceroute` — confirm if the change dropped packets before reaching the destination vs at the destination itself

**What they're testing:** Systematic elimination — DNS → port → TLS → routing → firewall — instead of guessing.

### Scenario 8: You inherit an undocumented legacy server. Build a network picture in 10 minutes.

```bash
hostname                  # what does it think it is
ip addr                   # interfaces, IPs
ip route                  # gateway, static routes
ss -tulnp                 # what's listening, on what ports
ip neigh                  # who it's recently talked to on L2
iptables -L -n -v         # any firewall rules in place
dig $(hostname)           # does DNS agree with what it thinks it is
```

**What they're testing:** Whether you have a repeatable, fast triage checklist rather than randomly poking around — the single most common "prove you're senior" question.

---

*Compiled as an interview prep reference for senior DevOps / SRE roles.*
