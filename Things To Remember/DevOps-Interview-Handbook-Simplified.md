# DevOps Interview Handbook (Simplified Edition)

A simple-English, easy-to-revise version of the Ultimate DevOps Interview Handbook — same technical depth, easier wording, with **memory tricks** added wherever a question is commonly confused.

> 💡 Wherever a "Trick" is given, it's a quick way to remember the answer under interview pressure.

---

## Table of Contents
1. [Linux](#1-linux)
2. [Networking](#2-networking)
3. [AWS](#3-aws)
4. [Docker](#4-docker)
5. [Kubernetes](#5-kubernetes)
6. [Terraform](#6-terraform)
7. [Ansible](#7-ansible)
8. [CI/CD & Jenkins](#8-cicd--jenkins)
9. [Git & GitHub](#9-git--github)
10. [Monitoring & Observability](#10-monitoring--observability)
11. [Logging](#11-logging)
12. [Security](#12-security)
13. [Site Reliability Engineering (SRE)](#13-site-reliability-engineering-sre)
14. [Production Scenarios](#14-production-scenarios)
15. [AI & Cloud](#15-ai--cloud)

---

## 1. Linux

**Q1. What is Linux?**
Linux is a free, open operating system kernel (started by Linus Torvalds in 1991). A "distro" like Ubuntu or CentOS bundles the kernel with tools and package managers so you get a full OS. Almost every server, container, and cloud VM runs on it.

**Q2. Explain Linux architecture.**
Layers, top to bottom:
```
User → Applications → Shell → Kernel → Hardware
```
- **User** – the person using the machine (e.g., you SSH-ing into a server)
- **Applications** – Docker, Jenkins, Nginx, etc. — they never talk to hardware directly
- **Shell** – the interface (Bash, Zsh) that passes your commands to the kernel
- **Kernel** – the core; handles processes, memory, CPU, devices, files, networking
- **Hardware** – CPU, RAM, disk, network card

> 💡 **Trick:** "Shell talks to the Kernel, Kernel talks to the Hardware." One line, covers both Q2 and the Kernel-vs-Shell follow-up.

**Q3. What happens when a Linux server boots?**
```
Power On → BIOS/UEFI → GRUB (bootloader) → Kernel → systemd/init → Services → Login Prompt
```
BIOS checks hardware and hands off to GRUB, GRUB loads the kernel, the kernel mounts the filesystem and starts the first process, systemd then starts services (SSH, Docker, cron), and finally you get a login prompt.

**Q4. What is a Process?**
A running copy of a program. Each one has a PID, a parent PID (PPID), and its own CPU/memory usage. Check with `ps -ef` or `top`.

**Q5. Process vs Thread**
| Process | Thread |
|---|---|
| Independent unit, own memory | Part of a process, shares memory |
| Slower to create | Faster to create |
| A crash usually stays isolated | A crash can bring down the whole process |

> 💡 **Trick:** Chrome = many processes (tabs), each process = many threads (rendering/network/JS).

**Q6. What is a Zombie Process?**
A process that finished running, but its parent hasn't "read" its exit status yet, so its entry lingers in the process table.
> 💡 **Trick:** Think of it as an ex-employee whose HR file is still open.

**Q7. What is an Orphan Process?**
A process whose parent died before it did. `systemd`/`init` adopts it and keeps it running.

| Zombie | Orphan |
|---|---|
| Already finished | Still running |
| Parent hasn't cleaned it up | Parent no longer exists |

**Q8. File Permissions**
Every file has three permission sets: Owner, Group, Others — each can have `r` (read), `w` (write), `x` (execute).
`-rwxr-xr--` → owner: full control, group: read+execute, others: read only.

**Q9. chmod 755**
`7 = rwx` (owner, full), `5 = r-x` (group), `5 = r-x` (others). Common for executable scripts — owner can edit, everyone else can run/read but not modify.
> 💡 **Trick:** Add the digits: r=4, w=2, x=1. 7=4+2+1(rwx), 5=4+1(r-x).

**Q10. Hard Link vs Soft Link**
- **Hard link** → points to the actual data (inode). Still works if the original name is deleted. Can't cross filesystems.
- **Soft link** → points to a path/filename (like a shortcut). Breaks if original is deleted. Can cross filesystems.

**Key commands:** `pwd`, `ls -la`, `cd`, `mkdir`, `touch`, `cp`, `mv`, `rm -rf`, `cat`, `less`, `grep`, `find`, `df -h`, `du -sh`, `free -m`, `top`/`htop`, `ps -ef`, `systemctl status`, `journalctl`

**Scenario: Server is slow — how do you debug?**
Structured order: `top` (CPU) → `free -m` (memory) → `df -h` (disk) → `du -sh /*` (find big folders) → `ps -ef` (processes) → `journalctl -xe` (logs) → `systemctl status <service>` (confirm key services are healthy).
> 💡 **Trick:** Memorize the order as **C-M-D-P-L-S** (CPU, Memory, Disk, Processes, Logs, Services) — never guess, always check top-down.

**Common mistakes:** memorizing commands with no context, jumping to `kill -9` instead of a graceful stop, mixing up hard/soft links, skipping logs, running `rm -rf` without double-checking the path.

---

## 2. Networking

**Q1. What is Networking?**
Connecting devices so they can exchange data — your browser talking to Google, Jenkins pulling from GitHub, Kubernetes pods talking to each other.

**Q2. OSI Model (7 layers)**
```
Application → Presentation → Session → Transport → Network → Data Link → Physical
```
- **Application (7):** HTTP, HTTPS, DNS — what the user actually sees
- **Presentation (6):** encryption, compression (e.g., TLS)
- **Session (5):** keeps you logged in
- **Transport (4):** TCP/UDP, ports, reliability
- **Network (3):** IP addresses, routers
- **Data Link (2):** MAC addresses, switches
- **Physical (1):** cables, Wi-Fi signals

> 💡 **Trick:** Don't memorize all 7 in isolation — if asked "what happens when you open google.com?", just walk top-down through this list.

**Q3. TCP/IP Model**
Simplified 4-layer version used in the real world: `Application → Transport → Internet → Network Access`. (OSI is mostly a teaching model.)

**Q4. TCP vs UDP**
| TCP | UDP |
|---|---|
| Connection-oriented (handshake first) | Connectionless |
| Reliable, guarantees delivery & order | Best-effort, no guarantee |
| Slower, more overhead | Faster, lightweight |
| Used by: HTTP, HTTPS, SSH, FTP | Used by: DNS, video streaming, gaming, VoIP |

> 💡 **Trick:** "TCP is a phone call (you confirm the other side is listening), UDP is throwing a postcard (fire and hope)." Use UDP when **speed > guaranteed delivery** (gaming, live video).

**Q5. What is an IP Address?**
A unique address for a device on a network — like a house number for your computer.

**Q6. Public vs Private IP**
- **Public** – reachable from the internet (e.g., an EC2 public IP)
- **Private** – only inside a private network (`10.x.x.x`, `172.16.x.x`, `192.168.x.x`)

**Q7. DNS + record types**
DNS turns `google.com` into an IP address.
- **A** – domain → IPv4
- **AAAA** – domain → IPv6
- **CNAME** – alias, one domain → another domain
- **MX** – mail server for the domain
- **TXT** – arbitrary text, used for verification/SPF/DKIM/DMARC
- **NS** – which name servers are authoritative for the domain

**Q8. HTTP vs HTTPS**
HTTP = plain text, port 80. HTTPS = encrypted via TLS, port 443. Always use HTTPS in production.

**Q9–10. SSL/TLS and the handshake**
TLS encrypts client-server traffic so passwords/tokens can't be intercepted.
```
Client → Hello → Server → Certificate → Key Exchange → Encrypted Communication
```

**Q11. What is a Port + important ports**
A port identifies which service on a machine you want (so one server can run SSH, Nginx, and Jenkins at once).
| Port | Service |
|---|---|
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet (insecure) |
| 25 | SMTP |
| 53 | DNS |
| 80 | HTTP |
| 110 | POP3 |
| 143 | IMAP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | Jenkins |
| 9090 | Prometheus |

> 💡 **Trick:** Group them — "insecure old stuff" (21, 23), "web" (80/443), "mail" (25/110/143), "databases" (3306/5432/6379), "tools" (22, 8080, 9090). Easier to recall in clusters.

**Q12. Load Balancer**
Spreads incoming traffic across many servers instead of hitting one server → better uptime, fault tolerance, performance.
- **Layer 4** – routes by IP/port, fast, simple
- **Layer 7** – routes by HTTP/URL/cookies, smarter/slower

**Q13. Reverse Proxy**
Sits in front of your app; clients think they're talking to it directly (e.g., Nginx). Gives you SSL termination, caching, load balancing, security.

**Q14. NAT**
Lets private IPs reach the internet without being directly exposed to it (AWS NAT Gateway does this for private subnets).

**Q15. CIDR**
A way of writing IP ranges, e.g. `192.168.1.0/24` = 256 addresses. Common in AWS VPC design.

**Scenario: `https://company.com` isn't loading — troubleshooting order**
`nslookup` (DNS) → `ping` (basic reachability) → `curl -I` (HTTP response) → `openssl s_client` (SSL cert) → check load balancer health → check app logs → check backend status → check firewall/security group rules.

**Key tools:** `ping`, `traceroute`, `telnet`, `curl`, `wget`, `nslookup`, `dig`, `ip addr`, `route`, `ss -tuln`, `netstat -tulnp`, `tcpdump`

**Common mistakes:** thinking ping tests HTTP (it only tests ICMP), confusing port 80 vs 443, mixing public/private IPs, thinking DNS stores website content (it only maps names→IPs).

---

## 3. Amazon Web Services (AWS)

**Global hierarchy:** `AWS Cloud → Region → Availability Zone (AZ) → Data Center`. A Region is a geographic area (e.g., `us-east-1`); an AZ is one or more physically separate data centers inside that region. Spreading resources across AZs = higher availability.

**Q1. Cloud Computing**
Renting compute/storage/networking over the internet instead of buying hardware — pay-as-you-go, scalable, highly available.

**Q2. Service models**
- **IaaS** – you manage OS/app/data, AWS manages hardware (e.g., EC2)
- **PaaS** – AWS also manages OS/runtime, you just deploy code (e.g., Elastic Beanstalk)
- **SaaS** – provider manages everything (Gmail, Salesforce)

**Q3–4. EC2 & AMI**
EC2 = a virtual server you can create/resize/stop/terminate. Before launching one you choose: AMI, instance type, VPC, subnet, security group, key pair, storage, IAM role.
An **AMI** (Amazon Machine Image) is the template used to launch an EC2 (OS + packages + config) — one AMI can launch hundreds of identical servers.

**Q5. AMI vs Snapshot**
AMI = launches new EC2 servers. Snapshot = backup of an EBS volume, used for recovery.

**Q6–7. EBS vs Instance Store**
EBS = persistent network storage that survives a reboot/stop (like the server's hard disk). Instance Store = temporary local storage, wiped on termination.

**Q8. S3**
Object storage for files (images, videos, backups, logs, ML models). Storage classes trade cost vs speed: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier (Instant/Flexible Retrieval), Glacier Deep Archive.

**Q9. S3 Versioning**
Keeps multiple versions of the same object — protects against accidental deletion/overwrite. Commonly used for Terraform state files.

**Q10. IAM**
Identity and Access Management — controls who can do what in AWS.
- **User** – one person/app
- **Group** – collection of users
- **Role** – temporary permissions attached to a resource (e.g., EC2 accessing S3)
- **Policy** – JSON document defining allow/deny permissions

> 💡 **Trick (Role vs Access Key):** Roles give **temporary, auto-rotated** credentials — safer than long-lived access keys, which can leak. If asked "why prefer roles?", answer: *temporary + auto-rotated = smaller blast radius.*

**Q11–12. VPC & Subnets**
VPC = your own isolated network in AWS, containing subnets, route tables, an Internet Gateway, NAT Gateway, security groups, and Network ACLs.
- **Public subnet** – reaches internet via Internet Gateway (load balancers, bastion hosts)
- **Private subnet** – no direct internet access (databases, backend services, K8s worker nodes)

**Q13. Security Group vs Network ACL**
| Security Group | Network ACL |
|---|---|
| Instance level | Subnet level |
| Stateful | Stateless |
| Allow rules only | Allow AND deny rules |

> 💡 **Trick:** "Security Group = firewall for the EC2 instance itself. NACL = firewall for the whole subnet."

**Q14–15. Internet Gateway vs NAT Gateway**
Internet Gateway = lets public resources be reached from the internet. NAT Gateway = lets *private* resources reach *out* to the internet (updates, APIs) without being reachable *from* it.

**Q16. Elastic Load Balancer (ELB)**
Distributes traffic across servers. Types: Application Load Balancer (ALB, most common for web apps), Network Load Balancer (NLB), Gateway Load Balancer.

**Q17. Auto Scaling**
Automatically adds/removes EC2 instances based on demand (e.g., scale up during a traffic spike, scale down at night) → saves cost and keeps performance stable.

**Q18. Route 53**
AWS's DNS service — maps domains to IPs, supports health checks, failover, weighted, and latency-based routing.

**Q19–20. CloudWatch vs CloudTrail**
CloudWatch = monitors metrics/logs and can trigger alarms (CPU, memory, network, disk). CloudTrail = records *who did what* via the AWS API — used for auditing/security investigations.
> 💡 **Trick:** "CloudWatch watches performance. CloudTrail tracks people."

**Q21. Lambda**
Run code without managing servers; it only runs when triggered (S3 upload, API Gateway, CloudWatch event, SNS). You pay only for execution time.

**Scenario: App is unreachable — how do you troubleshoot?**
Route 53 DNS → ALB health → Security Group rules → NACL rules → EC2 running? → app logs → CloudWatch metrics → target group health → app listening on the right port.

**Other core services to know:** EKS (managed Kubernetes), ECR (Docker image registry), RDS (managed relational DB), Secrets Manager, KMS (encryption keys).

**Common mistakes:** using the root account daily, giving every IAM user `AdministratorAccess`, committing AWS keys to Git, public databases, ignoring CloudWatch alarms, leaving unused resources running, forgetting encryption.

---

## 4. Docker

**Q1. What is Docker?**
An open-source tool that packages an app plus all its dependencies into a lightweight, portable **container** — so it behaves the same on your laptop, test server, and production. It ends the "works on my machine" problem.

**Q2. Containerization**
Bundling source code, libraries, runtime, and config into one isolated, portable package.

**Q3. VM vs Container**
| VM | Container |
|---|---|
| Full guest OS | Shares host OS kernel |
| Large, slow to start | Lightweight, starts in seconds |
| Needs a hypervisor | Needs Docker Engine |

> 💡 **Trick:** "Containers are light because they *share* the host kernel — a VM carries its own whole OS on its back."

**Q4. Docker Architecture**
```
Docker Client → Docker Daemon → Docker Registry
```
- **Client** – where you type `docker run nginx`
- **Daemon** – does the real work: builds images, runs containers, manages networks/volumes
- **Registry** – stores images (Docker Hub, Amazon ECR, ACR, Artifact Registry)

**Q5–6. Image vs Container**
Image = read-only blueprint. Container = a running instance of that image. One image → many containers (e.g., one Ubuntu image → 100 independent containers).

**Q7–8. Dockerfile & key instructions**
```dockerfile
FROM python:3.12
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["python","app.py"]
```
- `FROM` – base image (always first)
- `WORKDIR` – sets working directory (like `cd`)
- `COPY` – copies files into the image
- `RUN` – executes a command *during build* (e.g., installing packages)
- `CMD` – the default command when the container *starts* (only one per Dockerfile)
- `EXPOSE` – documents the port (doesn't actually publish it)

**RUN vs CMD vs ENTRYPOINT**
| RUN | CMD | ENTRYPOINT |
|---|---|---|
| Runs at build time | Runs when container starts | Defines the fixed main command |
| Creates image layers | Default startup command | Harder to override |
| For installing things | For starting the app | For fixed commands |

> 💡 **Trick:** "RUN builds it, CMD starts it, ENTRYPOINT locks it."

**Q9–10. Volumes & Bind Mounts**
Containers are temporary — delete the container, lose the data — unless you use a **volume** (managed by Docker, good for production) or a **bind mount** (maps a host folder, good for local dev).

**Q11. Docker Networking**
- **Bridge** – default, containers on the same bridge can talk to each other
- **Host** – container shares the host's network directly (faster, no isolation)
- **Overlay** – spans multiple Docker hosts (used in Swarm)

**Q12. Multi-stage builds**
Use one stage to build (e.g., Maven compiling Java) and copy only the final artifact into a slim final image — smaller, faster, more secure images since build tools don't ship to production.

**Q13. Why smaller images matter**
Faster download/upload, lower storage cost, faster CI/CD, smaller attack surface.

**Q14. Docker Compose**
Defines and runs multi-container apps from one YAML file: `docker compose up` instead of many separate `docker run` commands.

**Scenario: Container starts then immediately exits**
`docker ps -a` → `docker logs <id>` → `docker inspect <id>` → check Dockerfile & startup command → check env vars → test image locally → rebuild if needed.
> 💡 **Trick:** Container logs are almost always the fastest way to the root cause — check them *before* anything else.

**Key commands:** `docker images`, `docker ps`/`docker ps -a`, `docker pull`, `docker build`, `docker run`, `docker stop/start/restart`, `docker rm`/`docker rmi`, `docker logs`, `docker exec -it`, `docker inspect`, `docker network ls`, `docker volume ls`, `docker compose up`

**Common mistakes:** running as root inside containers, huge images, secrets baked into the Dockerfile, no `.dockerignore`, using `latest` tag in production, skipping vulnerability scans.

---

## 5. Kubernetes

**Q1. What is Kubernetes?**
An open-source container orchestration platform (from Google) that automates deployment, scaling, load balancing, self-healing, and rolling updates — so you're not managing thousands of containers by hand.

**Why it exists:** with 1 container, manual management is fine. With 5,000 containers, you need automatic answers to: which server runs what, what happens on crash, how do we scale, how do we update with zero downtime, how do containers find each other. Kubernetes answers all of these.

**Q2. Architecture**
```
Control Plane (API Server, Scheduler, Controller Manager, ETCD)
        ↓
Worker Nodes (run the actual Pods)
```
Control Plane = the "brain" (scheduling/decisions). Worker Nodes = where your app actually runs.

**Q3. Pod**
The smallest deployable unit — one or more containers sharing network + storage. You never deploy a bare container; it always lives inside a Pod.

**Q4–6. ReplicaSet vs Deployment**
```
Deployment → ReplicaSet → Pods
```
- **ReplicaSet** – keeps a fixed number of identical Pods running (recreates a crashed Pod)
- **Deployment** – manages ReplicaSets, adds rolling updates & rollbacks. In production, you almost always create Deployments, not ReplicaSets directly.

**Q7. StatefulSet**
For apps needing a stable identity (MySQL, PostgreSQL, MongoDB, Kafka) — Pods get fixed names like `mysql-0`, `mysql-1`.

**Q8. DaemonSet**
Ensures one Pod runs on *every* node — used for things like log collectors (Fluent Bit) or monitoring agents (Node Exporter). New node joins → gets one automatically.

**Q9. Service & types**
Pods are temporary and their IPs change, so a **Service** gives one stable endpoint.
- **ClusterIP** – default, internal only
- **NodePort** – exposes a port on every node (mostly for testing)
- **LoadBalancer** – provisions a cloud load balancer (production)
- **Headless** – used with StatefulSets for direct Pod discovery

> 💡 **Trick:** "NodePort = manual/testing, LoadBalancer = automatic/production."

**Q10. Ingress**
Routes HTTP/HTTPS traffic using paths/hosts through *one* load balancer instead of creating a separate one per service — cheaper, centralized, handles SSL termination.

**Q11–12. ConfigMap vs Secret**
| ConfigMap | Secret |
|---|---|
| Non-sensitive config (e.g. `LOG_LEVEL`) | Sensitive data (passwords, keys, tokens) |

**Q13–15. PV, PVC, StorageClass**
```
Application → PVC → PV → actual Storage
```
Persistent Volume (PV) = durable storage that survives Pod death. PVC = an app's *request* for that storage (apps use PVCs, not PVs directly). StorageClass = auto-provisions storage on demand instead of manually creating disks.

**Q16. HPA (Horizontal Pod Autoscaler)**
Automatically changes Pod count based on metrics (e.g., CPU > 70% → scale from 3 to 6 Pods; traffic drops → scale back down).

**Q17. RBAC**
Role-Based Access Control — governs who can do what inside the cluster (e.g., a developer can view Pods but not delete the cluster).

**Q18. Namespace**
Logically separates environments/apps (e.g., `production`, `staging`, `development`) — same resource name can exist in different namespaces.

**Q19–20. Self-Healing & Rolling Updates**
Self-healing: Pod crashes → ReplicaSet notices → creates a new one → app stays available. Rolling updates: new version replaces old Pods one at a time → zero downtime.

**Production issues — quick reference**
| Issue | Meaning | Likely causes |
|---|---|---|
| **CrashLoopBackOff** | App starts, crashes, restarts, repeat | App error, missing config, wrong env vars, DB unreachable |
| **ImagePullBackOff** | Can't download the image | Wrong image/tag, auth failure, private registry access |
| **Pending Pod** | Can't be scheduled | Not enough CPU/memory, node selector mismatch, missing PV |
| **OOMKilled** | Container exceeded memory limit | Needs more memory or app optimization |
| **NodeNotReady** | Worker node stopped talking to control plane | Network issue, kubelet failure, resource exhaustion |

**Scenario: Pod keeps crashing**
`kubectl get pods` → `kubectl describe pod` → `kubectl logs` (add `--previous` if needed) → `kubectl get events` → check ConfigMaps/Secrets → check resource limits → validate image → redeploy only after finding the root cause (never restart blindly).

**Key commands:** `kubectl get pods/deployments/services`, `kubectl describe pod`, `kubectl logs`, `kubectl exec -it`, `kubectl apply -f`, `kubectl delete -f`, `kubectl rollout status`/`kubectl rollout undo`, `kubectl top pod/node`, `kubectl get events`, `kubectl config get-contexts`/`use-context`

**Common mistakes:** deploying bare Pods instead of Deployments, storing passwords in ConfigMaps, skipping readiness/liveness probes, no CPU/memory requests-limits, using `latest` tag, ignoring namespaces, debugging without checking logs/events first.

---

## 6. Terraform

**Q1. Infrastructure as Code (IaC)**
Managing/provisioning infrastructure with code files instead of clicking through a cloud console — gives you automation, consistency, version control, repeatability, fewer human errors.

**Q2. What is Terraform?**
An open-source, cloud-agnostic IaC tool by HashiCorp — supports AWS, Azure, GCP, Kubernetes, GitHub, Cloudflare, etc.

**Q3. Workflow**
```
Write Code → terraform init → terraform plan → terraform apply
```
- `init` – downloads providers/plugins/modules (run first, always)
- `plan` – preview only, nothing changes yet
- `apply` – actually creates/updates infra (after confirmation)
- `destroy` – deletes everything Terraform created

**Q4–5. Provider & Resource**
A **Provider** connects Terraform to a platform (`provider "aws" { region = "ap-south-1" }`). A **Resource** is an actual infra component you define (e.g., `resource "aws_instance" "web" {...}`).

**Q6–7. Variables & Outputs**
Variables make config reusable (change one value, update everywhere it's used). Outputs print useful info after deployment (e.g., the EC2 public IP).

**Q8. Terraform State**
Stored in `terraform.tfstate` — tracks resource IDs, dependencies, metadata. Terraform compares desired config vs current state before making changes.
> 💡 **Trick:** "Without the state file, Terraform doesn't know what already exists — it might try to recreate things and cause duplication or failure."

**Q9–10. Remote Backend & State Locking**
Keeping state on one laptop is risky — store it remotely (e.g., S3) for shared team access and backup. **State Locking** (commonly S3 + DynamoDB) prevents two engineers running `apply` at the same time from corrupting the state.

**Q11. Modules**
Reusable Terraform code blocks (e.g., a `vpc` module you reuse across projects) — improves maintainability, avoids duplication.

**Q12. count vs for_each**
| count | for_each |
|---|---|
| Identical resources | Resources need unique values |
| e.g., `count = 3` → 3 identical EC2s | e.g., `for_each = toset(["dev","stage","prod"])` |

> 💡 **Trick:** "Same thing repeated = count. Different values per item = for_each."

**Q13. depends_on**
Manually tells Terraform to wait for one resource before creating another, for cases it can't auto-detect.

**Q14. Workspaces**
Let you run multiple environments (dev/staging/prod) from the same code, each with its own state.

**Q15. terraform import**
Brings an already-existing (manually created) resource under Terraform's management instead of recreating it.

**Q16. Infrastructure Drift**
When someone changes a resource manually outside Terraform (e.g., resized an instance in the console) — `terraform plan` detects the mismatch.

**Scenario: Terraform deployment fails**
`terraform validate` → `terraform fmt` → `terraform plan` → check cloud credentials → check state file/remote backend connectivity → check cloud quotas. Never edit the state file by hand unless absolutely necessary (and always back it up first).

**Key commands:** `terraform init/validate/fmt/plan/apply/destroy/output/show`, `terraform state list`, `terraform import`, `terraform workspace list/select`

**Common mistakes:** committing `terraform.tfstate` to Git, keeping state only locally for team projects, hand-editing state, hardcoding credentials, skipping `plan` before `apply`, no state locking in team environments.

---

## 7. Ansible

**Q1. Configuration Management**
Keeping many servers in the same, predictable state (same packages, same users, same permissions) instead of configuring each one by hand.

**Q2. What is Ansible?**
An open-source automation/config-management tool (Red Hat) that installs software, configures servers, manages users/services — **agentless** (no software needed on target machines).

**Q3. Architecture**
```
Control Node → SSH → Managed Nodes → Task Execution
```
Control Node = where Ansible runs and executes playbooks. Managed Nodes = the servers being configured — reachable purely via SSH, no agent install needed.

**Q4. Inventory**
The list of servers Ansible manages, grouped (e.g., `[web]`, `[database]`), so you don't specify IPs every time.

**Q5. Playbook**
A YAML file describing automation tasks, e.g. installing Nginx on every server in the `web` group. YAML is used because it's human-readable and simple.

**Q6. Modules**
Reusable task units instead of raw shell scripts: `apt`/`yum` (install packages), `service`, `copy`, `file`, `user`, `git`, `command`, `shell`.

**Q7. Idempotency**
Running the same playbook multiple times gives the same result — e.g., if Docker is already installed, Ansible does nothing (doesn't reinstall). This makes automation safe — no unnecessary restarts/reinstalls.

**Q8. Variables**
Make playbooks reusable — change one variable value, everything using it updates.

**Q9. Handlers**
Tasks that run *only when notified* — e.g., restart Nginx only if its config actually changed. Avoids unnecessary restarts/downtime.

**Q10. Roles**
Organize large playbooks into reusable folders (tasks, variables, templates, handlers, files) — e.g., a `nginx` role, a `docker` role. Standard in enterprise projects.

**Q11. Ansible Vault**
Encrypts sensitive data (passwords, API keys) so secrets stay safe even if the repo is shared.

**Q12. Templates**
Generate config files dynamically using Jinja2 (e.g., `ServerName {{ hostname }}` — each server gets its own value automatically).

**Q13. command vs shell**
| command | shell |
|---|---|
| Executes directly, more secure | Goes through the shell, supports pipes/redirection |
| Preferred by default | Use only when you need shell features |

**Q14. Ad-hoc commands**
One-off tasks without a full playbook, e.g. `ansible all -m ping` (test connectivity) or `ansible web -m apt -a "name=nginx state=present"`.

**Scenario: Install Docker on 200 servers**
Add all servers to inventory → write a Docker-install playbook → use package modules → start/enable the service → validate → run the playbook against all hosts at once. Far faster and more consistent than logging into each one.

**Key commands:** `ansible --version`, `ansible all -m ping`, `ansible-playbook playbook.yml`, `ansible-inventory --list`, `ansible-vault create/edit`, `ansible-galaxy init/install`

**Common mistakes:** hardcoded passwords in playbooks, overusing `shell` instead of proper modules, one giant playbook instead of roles, ignoring idempotency, skipping Vault for secrets.

---

## 8. CI/CD & Jenkins

**Q1. CI/CD**
- **Continuous Integration (CI)** – every code commit triggers automatic build + test + static analysis → catch problems early
- **Continuous Delivery** – app is auto-prepared for release, but production deploy still needs manual approval
- **Continuous Deployment** – fully automatic; passing all stages = auto-deploy to production

> 💡 **Trick:** "Delivery = ready, but a human presses go. Deployment = no human needed at all."

**Typical pipeline flow:**
```
Git Push → GitHub → Jenkins → Build → Test → Security Scan → Docker Build → Deploy
```

**Q2–3. Jenkins & Architecture**
Jenkins is an open-source automation server for build/test/deploy across almost any stack. It uses a **Controller (Master) + Agent** architecture: the Controller schedules jobs/manages plugins/credentials; Agents actually do the work (compile, test, build images, deploy). Using agents = faster builds, parallel execution, better scaling.

**Q4–5. Pipeline & Jenkinsfile**
A Pipeline defines the CI/CD workflow *as code* in a **Jenkinsfile** stored in Git — gives you version control, reproducibility, easier maintenance.
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { echo 'Building...' }
        }
    }
}
```

**Q6. Typical stages**
`Checkout → Build → Test → Code Quality → Security Scan → Docker Build → Push Image → Deploy`

**Q7. Declarative vs Scripted Pipeline**
| Declarative | Scripted |
|---|---|
| Simpler, easier to maintain | More flexible, needs Groovy |
| Preferred for most projects | Used for complex workflows |

**Q8. Webhook**
Notifies Jenkins the instant new code is pushed, instead of Jenkins polling GitHub every minute — faster and less wasteful.

**Q9–10. Credentials**
Never hardcode AWS keys or passwords in a Jenkinsfile — store them in Jenkins Credentials and retrieve them securely at runtime.

**Q11. Shared Libraries**
If 20 projects share the same deployment logic, put it in one Shared Library instead of copy-pasting the pipeline everywhere — reusable, easier to maintain, standardized.

**Q12. Plugins**
Extend Jenkins (Git, Docker, Kubernetes, Pipeline, Blue Ocean, SonarQube, Slack, etc.).

**Q13. A realistic production pipeline**
```
Git Checkout → Compile → Unit Tests → SonarQube → Dependency Scan → Docker Build →
Trivy Scan → Push to ECR → Update Helm Chart → Argo CD Deploy → Smoke Test
```
Notice deployment is just one stage among many.

**Q14–15. Failure handling & rollback**
A good pipeline stops immediately on failure, notifies the team, keeps logs, and blocks deployment — production should never get a failed build. Rollback options: previous Docker image, previous Helm release, previous Git commit, or an Argo CD rollback.

**Scenario: Pipeline suddenly fails**
Read console output → find the failing stage → check Git connectivity → verify credentials → confirm tools (Docker/Maven/Python) are available → check disk space → only retry after understanding *why* it failed.

**Key commands (on the Jenkins host):** `systemctl status/restart jenkins`, `journalctl -u jenkins`, `docker logs jenkins`, `java -jar jenkins-cli.jar`

**Common mistakes:** hardcoded credentials in the Jenkinsfile, running everything on the Controller instead of Agents, skipping tests/security scans, unnecessary plugins, giving Jenkins admin access to cloud accounts, not version-controlling the Jenkinsfile.

---

## 9. Git & GitHub

**Q1. What is Git?**
A distributed version control system (Linus Torvalds) that tracks source code history over time — lets developers collaborate safely, roll back, branch, and merge.

**Q2. Why version control?**
Without it, multiple developers editing the same file can silently overwrite each other's work with no record of who/what/why changed.

**Q3. Git's flow**
```
Working Directory → Staging Area (git add) → Local Repository (git commit) → Remote Repository (git push)
```

**Q4. Git vs GitHub**
| Git | GitHub |
|---|---|
| Version control tool | Cloud hosting platform |
| Installed locally, works offline | Hosts repos, needs internet |

> 💡 **Trick:** "Git manages versions. GitHub hosts them."

**Q5–6. Repository & Commit**
A **repository** stores code, history, branches, and tags. A **commit** is a snapshot of your project at a point in time, identified by a unique hash.

**Q7. Branch**
Lets developers work independently without disturbing the main codebase; branches get merged once work is done.

**Q8. Merge vs Rebase**
| Merge | Rebase |
|---|---|
| Preserves history | Rewrites history (linear) |
| Creates a merge commit | No extra merge commit |
| Safer for shared branches | Best for local/feature branches |

**Q9. Pull Request (PR)**
A request to merge one branch into another — used for code review before merge.

**Q10. git clone**
Downloads an existing repository as a local copy.

**Q11. fetch vs pull**
| fetch | pull |
|---|---|
| Downloads changes only | Downloads + merges |
| Safe, review before merging | Immediately updates your branch |

**Q12. git stash**
Temporarily saves unfinished work (e.g., to handle an urgent bug) with `git stash`, then restore it later with `git stash pop`.

**Q13. reset vs revert**
| reset | revert |
|---|---|
| Moves branch pointer backward, rewrites history | Adds a new commit that undoes changes, history intact |
| Good for local work | **Safer for shared/production repos** |

> 💡 **Trick:** "Reset erases history, revert adds to it. In production, always prefer revert."

**Q14. git cherry-pick**
Copies one specific commit from another branch instead of merging the whole branch.

**Q15. GitOps**
Git becomes the single source of truth: `Git Push → GitHub → Argo CD → Kubernetes` — nobody deploys manually; production always matches what's in Git.

**Scenario: Merge conflict between two developers**
`git status` → open the conflicted file → compare both versions → keep the right code, remove conflict markers → `git add .` → `git commit`. Never blindly accept one side — understand the conflict first.

**Key commands:** `git init/clone/status/add/commit/log`, `git branch/checkout/switch`, `git merge/rebase`, `git fetch/pull/push`, `git stash`/`git stash pop`, `git reset/revert/cherry-pick`, `git tag`

**Common mistakes:** committing straight to `main`, vague commit messages ("update", "fix"), force-pushing to shared branches without understanding the impact, confusing fetch/pull, using reset on shared repos instead of revert.

---

## 10. Monitoring & Observability

**Q1–2. Monitoring vs Observability**
Monitoring collects and shows metrics — tells you *something* is wrong. Observability combines **Metrics + Logs + Traces** to explain *why*.

| Monitoring | Observability |
|---|---|
| Detects problems | Explains problems |
| Predefined metrics | Metrics + logs + traces |
| Answers "what happened?" | Answers "why did it happen?" |

**Q3–4. Prometheus**
An open-source monitoring tool that pulls (scrapes) metrics from apps/infra at intervals and stores them as time-series data.
```
Application → /metrics endpoint → Prometheus → stores metrics → Grafana dashboard
```

**Q5. Exporter**
Converts a system's raw data into Prometheus-readable metrics. Examples: Node Exporter (Linux host metrics), MySQL/PostgreSQL exporters, Blackbox Exporter, cAdvisor.

**Q6. Grafana**
A visualization tool — connects to Prometheus and *displays* dashboards. It does not collect metrics itself.

| Prometheus | Grafana |
|---|---|
| Collects metrics | Displays metrics |
| Time-series storage | Dashboards |

> 💡 **Trick:** "Prometheus collects. Grafana presents."

**Q7. PromQL**
Prometheus's query language, e.g. `up` (is the target alive?) or `node_cpu_seconds_total` (CPU usage).

**Q8. Alertmanager**
Sends notifications (Slack, email, PagerDuty, webhooks) when Prometheus alert rules fire: `Prometheus → Alert Rule → Alertmanager → Engineer`.

**Q9. What to monitor**
- Infra: CPU, memory, disk, network
- Application: request rate, error rate, response time
- Kubernetes: Pod/Node status, restart counts, container memory
- Business: active users, orders/min, payment success rate

**Q10. RED Method (for APIs/microservices)**
**R**ate (requests/sec), **E**rrors (failed requests), **D**uration (response time).

**Q11. USE Method (for infrastructure)**
**U**tilization, **S**aturation, **E**rrors — e.g., for CPU: utilization 85%, saturation = long queue, errors = throttling.

**Q12. A good alert**
Actionable, specific, relevant, easy to understand. Bad: "CPU High." Good: "Production Node-3 CPU has stayed above 90% for 10 minutes."

**Scenario: App suddenly slow**
Check Grafana dashboards → CPU/memory → response time → error rate → Pod health → app logs → DB performance → recent deployments — find the bottleneck, don't guess.

**Components:** Prometheus (collects), Grafana (visualizes), Alertmanager (notifies), Node Exporter (host metrics), cAdvisor (container metrics), kube-state-metrics (K8s object state).

**Common mistakes:** monitoring infra but not the app, too many alerts causing alert fatigue, unrealistic thresholds, ignoring trends, dashboards that are hard to read, checking metrics without checking logs.

---

## 11. Logging

**Q1–2. Logging & why it matters**
Logging records events (from apps, OS, containers) so you know *what* happened, *when*, *where*, and *why*. Without logs, "the payment failed" gives you zero clues about which server/user/cause.

**Q3. Centralized Logging**
Instead of searching logs on each of 300 pods individually, collect everything into one platform:
```
Application → Container Logs → Log Collector → Centralized Platform → Search & Analysis
```

**Q4. ELK Stack**
- **Elasticsearch** – stores/indexes logs (a search engine for log data)
- **Logstash** – collects and transforms logs
- **Kibana** – UI to search, filter, and visualize

**Q5–6. EFK Stack & Fluent Bit**
EFK swaps Logstash for **Fluentd** (lighter, popular in Kubernetes). **Fluent Bit** is an even lighter collector, often run as a DaemonSet on every node.

**Q7. Loki**
A logging tool from Grafana Labs that indexes only metadata/labels (not the whole log body) → cheaper storage, faster setup, integrates naturally with Grafana.

| ELK | Loki |
|---|---|
| Indexes full log content | Indexes only labels/metadata |
| More powerful search | Lighter, cheaper |

**Q8. Structured Logging**
Instead of a vague `"Something went wrong"`, log structured details: timestamp, level, service, user ID, error, order ID — much easier to search.

**Q9. Correlation ID**
One request may pass through 5 microservices; a Correlation ID ties all their log entries together so you can trace one request across the whole system instead of searching each service separately.

**Q10. Log Rotation**
Archives/compresses/deletes old logs automatically so disks don't fill up and cause outages.

**Scenario: Orders failing, infra looks healthy**
Check application logs for the affected service → search errors around the reported time → use the Correlation ID to trace across microservices → if DB errors appear, check DB connectivity → if API failures appear, check the dependent service. Follow the whole request path, don't guess.

**Key commands:** `kubectl logs <pod>` / `kubectl logs -f <pod>`, `docker logs <id>`, `journalctl -xe`, `grep ERROR file.log`, `grep -R "text" /var/log`

**Common mistakes:** logging secrets/passwords, vague messages, ignoring timestamps, no log rotation, logs only on local disk (not centralized), excessive debug logging in production.

---

## 12. Security

**Q1. DevSecOps**
Security built into *every stage* of the pipeline (not just before production):
```
Developer → Git Push → Build → Security Scan → Docker Build → Container Scan → Deploy
```

**Q2. Principle of Least Privilege (PoLP)**
Give users/apps/services only the permissions they actually need — if compromised, the damage is limited to what that account could do anyway.

**Q3–4. IAM & RBAC**
IAM controls AWS-level access (users/groups/roles/policies). RBAC controls Kubernetes-level access (e.g., a developer can view Pods/logs but not delete Deployments or modify namespaces).

**Q5. Secrets Management**
Never hardcode secrets in code or commit them to Git — even a deleted commit still lives in Git history, and attackers actively scan public repos for leaked secrets. Use Kubernetes Secrets, AWS Secrets Manager, or HashiCorp Vault instead.

**Q6. TLS**
Encrypts client-server traffic so passwords and card numbers can't be intercepted (HTTPS = HTTP + TLS).

**Q7. Trivy**
Open-source vulnerability scanner for Docker images, Kubernetes, filesystems, and Git repos — finds known CVEs, misconfigurations, and leaked secrets. Often run automatically in CI/CD before deploy.

**Q8–9. SAST vs DAST**
| SAST (Static) | DAST (Dynamic) |
|---|---|
| Scans source code, app *not* running | Tests a *running* app |
| Finds SQL injection risk, hardcoded creds, insecure code | Finds XSS, broken auth, SQL injection live |

> 💡 **Trick:** "SAST reads the code, DAST attacks the running app — use both together."

**Q10. Securing Docker images**
Use official base images, avoid `latest`, run as non-root, remove unnecessary packages, use multi-stage builds, scan for vulnerabilities, keep images small.

**Q11. Securing Kubernetes**
Enable RBAC, use Network Policies, store creds in Secrets, apply resource limits, scan images, restrict privileged containers, keep versions updated, enable audit logging.

**Scenario: A developer commits AWS keys to Git by mistake**
Treat the keys as compromised immediately → remove them from the repo → rotate/delete the exposed credentials → issue new ones → check CloudTrail for suspicious activity → update the app with new creds → add secret scanning to CI/CD → tell the team. Deleting from Git alone is **not enough** — history still contains them.

**Key commands:** `trivy image nginx:latest`, `kubectl get secrets`/`kubectl describe secret`, `aws iam list-users`, `kubectl get roles`/`kubectl get rolebindings`

**Common mistakes:** using the AWS root account daily, giving everyone `AdministratorAccess`, secrets in source code, containers running as root, ignoring scan results, delayed patching, shared credentials across a team.

---

## 13. Site Reliability Engineering (SRE)

**Q1. What is SRE?**
Applying software-engineering principles to operations to make systems reliable, scalable, and highly available — focused on *preventing* failures, not just reacting to them.

**Q2–4. SLA vs SLO vs SLI**
- **SLI (Indicator)** – the actual measurement (e.g., 99.97% uptime this month)
- **SLO (Objective)** – the internal target (e.g., 99.95%)
- **SLA (Agreement)** – the external promise to customers (e.g., 99.9%, often with penalties if missed)

> 💡 **Trick:** "SLI = what I measured. SLO = what I'm aiming for. SLA = what I promised the customer." SLOs are usually set *tighter* than SLAs, as a safety margin.

**Q5. Error Budget**
The acceptable amount of "imperfection" left after your SLO (e.g., SLO = 99.9% → 0.1% is your budget). If the team burns through the whole budget, focus shifts from new features to reliability work.

**Q6. High Availability (HA)**
The app keeps running even if one component fails — e.g., if Server B crashes behind a load balancer, traffic just shifts to A and C.

**Q7. Disaster Recovery (DR)**
The plan for restoring service after a major failure (data center outage, region outage, ransomware, hardware failure) — defines backup strategy, recovery steps, and responsibilities.

**Q8. RTO vs RPO**
- **RTO (Recovery Time Objective)** – *how fast* you must recover (e.g., within 30 minutes)
- **RPO (Recovery Point Objective)** – *how much data* you can afford to lose (e.g., no more than 5 minutes)

> 💡 **Trick:** "RTO = Time to recover. RPO = data you're okay losing (Point in time)." Just remember **T = Time, P = data Point**.

**Q9. Root Cause Analysis (RCA)**
Finding the *real* underlying cause, not just the symptom — e.g., app crashed → DB unavailable → disk full → log rotation was never configured. The real fix is log rotation, not restarting the app.

**Q10. Capacity Planning**
Estimating CPU/memory/storage/network/K8s-node needs *before* traffic grows, to avoid performance issues at peak load.

**Scenario: Outage during a big sales event**
Confirm via monitoring → assess user impact → notify stakeholders → identify affected services → collect logs/metrics → restore service ASAP → monitor stability → do an RCA afterward → document lessons → put preventive measures in place.
> 💡 **Trick:** "Restore first, investigate later." Never delay recovery to do deep investigation mid-incident.

**Common mistakes:** treating every alert as equally urgent, fixing symptoms not root causes, skipping documentation/post-incident reviews, shipping new features after the error budget is exhausted.

---

## 14. Production Scenarios

A quick-reference cheat sheet — interviewers care more about *your process* than the exact final answer.

| Scenario | Troubleshooting order |
|---|---|
| **Pod CrashLoopBackOff** | `kubectl get pods` → `describe pod` → `logs` (+ `--previous`) → check env vars/ConfigMaps/Secrets/DB connectivity/resource limits |
| **ImagePullBackOff** | Verify image name/tag exists → check registry credentials → check ImagePullSecret → check node internet access → `describe pod` for events |
| **Website returns 503** | DNS → Load Balancer → backend servers → Kubernetes Services → app logs (503 usually = app unavailable, not network down) |
| **Jenkins pipeline fails** | Identify failing stage → read console logs → check Git connectivity, credentials, Docker availability, dependencies, disk space |
| **Terraform state lock error** | Confirm no one else is running Terraform → check the DynamoDB lock → only remove it once confirmed safe (never force-delete blindly) |
| **EC2 unreachable via SSH** | Instance running? → Security Group allows port 22? → correct key pair? → NACL/Route Table/IGW/public IP? → SSH service running? → then OS logs |
| **High CPU** | `top`/`htop` to find the process → check recent deploys, infinite loops, memory pressure, DB queries, traffic spikes |
| **High memory** | `free -m` → `ps -aux --sort=-%mem` → check for leaks, big caches, JVM heap, container limits |
| **Disk full** | `df -h` → `du -sh /*` → check app logs, Docker images, container logs, temp files (log rotation is the usual culprit) |
| **SSL cert expired** | Check expiry → check auto-renewal (Let's Encrypt) → replace cert → reload web server → confirm HTTPS |
| **DB connection failure** | DB running? → credentials correct? → Security Group allows traffic? → network connectivity? → DB logs → connection limits |
| **Node NotReady** | `kubectl get nodes` → `describe node` → check kubelet, network, disk, CPU, memory → cordon/drain before maintenance |
| **Bad deployment** | Stop further deploys → rollback immediately (Helm release / Docker image / K8s Deployment / Git version) → THEN do RCA |
| **DNS failure** | Route 53 config → DNS records → name servers → TTL values → domain expiry → DNS propagation |
| **CPU alert fires** | Don't restart blindly → check Grafana → recent deploys → running processes → traffic patterns → autoscaling status |

**The universal troubleshooting mindset (memorize this flow):**
```
Understand the Problem → Collect Evidence → Review Metrics → Review Logs →
Identify Root Cause → Implement Fix → Verify Recovery → Document RCA → Prevent Recurrence
```
> 💡 **Trick:** If you're ever stuck on *any* scenario question, just narrate this 9-step flow applied to that scenario — interviewers are grading the process, not a perfect memorized fix.

**Common interview mistakes:** restarting services before checking logs, blaming the "most recent deployment" with no evidence, ignoring dashboards, deleting Pods instead of diagnosing, fixing without understanding root cause, skipping documentation.

---

## 15. AI & Cloud

**Q1–2. AI vs ML vs Generative AI**
AI = machines doing tasks that normally need human intelligence. Machine Learning = a subset of AI where systems learn from data instead of explicit rules. Generative AI = creates new content (text, images, code, audio) — e.g., ChatGPT.

**Q3. LLM**
Large Language Model — trained on massive text to understand/generate human-like language; can answer questions, write code, summarize, explain concepts (GPT-based models, Claude, Gemini, Llama, etc.).

**Q4. RAG (Retrieval-Augmented Generation)**
Instead of relying only on what it was trained on, the model *retrieves* relevant info from an external knowledge base first, then generates the answer:
```
Question → Knowledge Base → Relevant Docs Retrieved → LLM → Accurate Answer
```
Useful for answering from company docs/internal wikis.

**Q5. Amazon Bedrock**
A fully managed AWS service giving API access to foundation models — build Generative AI apps (chatbots, summarization, search) without managing infrastructure.

**Q6. AI Agents**
Unlike a simple chatbot, an Agent can plan multiple steps, use tools, and take action — e.g., told "deploy my application," it might check GitHub, trigger Jenkins, verify tests, deploy to Kubernetes, and report the result.

**Q7. MCP (Model Context Protocol)**
An open protocol letting AI models securely talk to external tools/data (GitHub, databases, file systems, cloud platforms, monitoring tools) — a standardized way for AI to access context and actually do work, not just answer questions.

**Q8. How AI helps DevOps**
Writing Terraform/K8s manifests, explaining CI/CD failures, summarizing incidents, spotting security issues, generating docs, reviewing PRs, suggesting cost optimizations. AI assists — it doesn't replace engineering judgment.

**Scenario: Manager wants AI in the DevOps workflow — where do you use it?**
Use it to boost productivity, not to auto-approve critical decisions: generate IaC templates, help build Jenkins pipelines, summarize logs, explain K8s errors, draft incident reports, review PRs, flag security issues, suggest infra optimizations. Production deploys should still require **human review and approval**.

**Common mistakes:** assuming AI replaces engineers, trusting AI-generated code without review, pasting sensitive company data into public AI tools, skipping validation of AI output, ignoring compliance/security when integrating AI.

---

## Final Interview Checklist

Before walking into an interview, make sure you can honestly say "yes" to:

- [ ] I can explain every tool here in plain language, not just definitions
- [ ] I can describe how these tools connect in a real CI/CD pipeline
- [ ] I can troubleshoot production failures with a structured process (not guesswork)
- [ ] I can explain trade-offs (not just "what is X")
- [ ] I can tie answers back to something I've actually built or worked on

> 💡 **Final trick:** Interviewers are testing *how you think*, not just what you can recite. When in doubt on any scenario question, fall back to: **Understand → Collect Evidence → Metrics → Logs → Root Cause → Fix → Verify → Document → Prevent.**

---

*Simplified and reorganized for personal revision — original source material adapted for clarity and quick recall.*
