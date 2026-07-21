# Senior DevOps Engineer Interview Prep

Distilled from 50+ real interview loops. This isn't about memorizing definitions — it's about proving you've actually operated infrastructure under pressure. Below: the patterns that separate candidates, the red/green flags, and a prep checklist with sample Q&A.

---

## 1. The Baseline Is Table Stakes

Everyone knows Docker, core CI/CD concepts, core AWS services, and surface-level Kubernetes (Deployments, ConfigMaps vs Secrets, HPA, PDB). This gets you in the door — it does **not** differentiate you. Interviewers can predict the "correct but forgettable" answer before you finish it. Assume the bar starts here, not ends here.

---

## 2. Where Most Candidates Fall Apart

### a) Kubernetes knowledge is wide but shallow
Candidates can define resources but freeze on *why* things happen.

- **Weak answer pattern:** jumping straight to "check the logs" without considering node pressure, eviction thresholds, or priority classes.
- **What to fix:** be ready to reason through failure mechanics, not just terminology.

### b) Nobody wants to talk about failure
The incident question is the single most revealing one asked, and most people flinch.

- **Weak pattern:** sanitized stories with no personal fault, or "we had some downtime, the team handled it."
- **What's wanted:** specificity, ownership, and a concrete change made afterward (runbook, architecture, alerting).

### c) Observability is a checklist, not a practice
"We use Prometheus and Grafana" is not a differentiator.

- **Weak pattern:** can't describe alerts they personally wrote, dashboards that caught a real problem, or how they separate noisy vs. meaningful metrics.
- **What's wanted:** an SLO mindset and lived experience tuning a system after an alert storm.

### d) No opinions on tradeoffs
Seniority means having a reasoned position, not reciting both sides neutrally.

- **Weak pattern:** endless hedging on questions like OpenShift vs. Kubernetes.
- **What's wanted:** "I'd lean toward X because of Y and Z, but that changes if the team has experience with..." — reasoning shown, not just knowledge recited.

### e) Vagueness about actual scope of ownership
This is called out as **the single thing most likely to kill an otherwise strong candidate.**

- **Weak pattern:** describing something (multi-region deployment, large Terraform setup) as personally owned, then failing detailed follow-ups because the role was actually adjacent or maintenance-only.
- **What's wanted:** precision. "I built the monitoring layer on a platform my team designed" beats an inflated ownership claim that collapses under scrutiny.

---

## 3. Green Flags That Make Candidates Stand Out

- **Volunteer failure stories** without being prompted — debugging and postmortems are just part of how they narrate their career.
- **Ask sharp, evaluative questions back**, e.g.:
  - "What's your current deployment frequency, and where does it get stuck?"
  - "What does your on-call rotation look like, and has it caused attrition?"
- **Connect tools to outcomes**, not just adoption. E.g.: "We moved to Kubernetes because deployment lead time was 45 minutes and needed to be under 10 — here's what we learned in the first three months," instead of just "we moved to Kubernetes."
- **Have opinions on Day-2 operations**: drift detection, secrets rotation, keeping Terraform state clean across teams. This is where strong candidates separate from average ones.

---

## 4. What Changed How the Interviewer Hires

- Post-incident questions are now weighted heavily — how you tell a failure story reveals engineering maturity better than most technical questions.
- Tool name-dropping ("Kubernetes" on a resume) is now a starting point, not a credential.
- How you handle **"I don't know"** matters a lot. Freezing is a red flag. The strong pattern: "I don't know off the top of my head, but here's how I'd think through it" — followed by actually reasoning it through live.
- Vague job descriptions attract vague candidates — so precise self-scoping is increasingly rewarded.

---

## 5. Prep Checklist

- [ ] Stop memorizing definitions — be ready to explain *when* you'd use something and what tradeoff you accepted.
- [ ] Prepare 2–3 specific, owned incident stories: timeline, root cause, fix, and what you personally changed afterward.
- [ ] Form real opinions on common tradeoffs (Terraform vs. Pulumi, Kubernetes vs. OpenShift, etc.) with reasoning — avoid "it depends" as a full answer.
- [ ] Be precise about scope of ownership on every project you mention. Don't claim more than you can defend under follow-up questions.
- [ ] Prepare sharp questions to ask the interviewer — this is itself evaluated as a signal of seriousness.

---

## 6. Sample Questions and How to Answer Them

### Q: "Walk me through what happens when a pod gets OOMKilled and the node is under memory pressure."
**What they're testing:** depth beyond definitions.
**Answer approach:** Explain the sequence — kubelet detects memory pressure via cgroup/node stats, evicts pods based on QoS class and priority, OOM killer may act at the container level if a limit is exceeded before node-level eviction triggers. Mention QoS classes (Guaranteed/Burstable/BestEffort) and how requests/limits affect eviction order.

### Q: "A deployment that's been stable for weeks suddenly has pods getting evicted. Where do you start?"
**What they're testing:** systematic incident triage, not just "check the logs."
**Answer approach:** Start with node-level signals (memory/disk pressure, eviction thresholds), check for recent changes (deploys, autoscaling events, noisy neighbors), then narrow to pod-level logs and events. Show a structured triage path, not a single reflexive action.

### Q: "Tell me about an incident you owned — what went wrong, how you handled it, what you changed after."
**What they're testing:** ownership and evidence of learning.
**Answer approach:** Use a real, specific incident. Structure: what broke → why it was missed → how you diagnosed and resolved it → the concrete change you made afterward (alert, runbook, architecture fix). Avoid deflecting blame to "the team."

### Q: "How do you think about OpenShift vs. Kubernetes for a new workload?"
**What they're testing:** whether you have a defensible opinion.
**Answer approach:** Take a position and justify it with tradeoffs (e.g., governance/built-in tooling vs. flexibility and ecosystem breadth), then note what would change your answer (team experience, compliance needs, existing infra).

### Q: "What alerts have you personally written, and what dashboards have actually caught a real problem?"
**What they're testing:** whether observability is a practice or a checklist for you.
**Answer approach:** Give a specific alert you authored, the threshold logic behind it, and a real incident it caught or a false-positive you tuned out. Speak in terms of SLOs where possible.

### Q: "Can you walk me through the most complex piece of infrastructure you've personally owned?"
**What they're testing:** honesty about scope — this is the highest-risk question.
**Answer approach:** Be exact about what you designed vs. maintained vs. contributed to. It's fine to say "I inherited this and extended X" — what erodes trust is claiming full ownership and then failing detailed follow-ups.

### Q: "What does drift detection / secrets rotation / Terraform state management look like in your current setup?"
**What they're testing:** Day-2 operations maturity beyond initial deployment.
**Answer approach:** Describe your actual process — tooling used, cadence, how state is shared/locked across teams, how rotation is automated or triggered. Concrete specifics beat naming a tool.

---

## 7. Additional Interview Question Bank (Technical)

### Q: "How would you design a CI/CD pipeline for a microservices application?"
**Model answer:** "I'd start by separating build, test, and deploy concerns per service so teams can ship independently. On push, run unit tests and lint, build a container image, and tag it with the commit SHA — not `latest` — so every deploy is traceable. Push to a registry like ECR, then run integration tests against a staging namespace before promoting. For deploys, I'd use a progressive strategy — canary or blue/green — with automated rollback tied to error-rate and latency SLOs rather than a manual gate. The part people skip is making rollback boring and automatic: if I can't roll back in under a minute without a human making a judgment call, the pipeline isn't done."

### Q: "What's the difference between a Deployment and a StatefulSet, and when would you choose each?"
**Model answer:** "A Deployment treats pods as interchangeable — no stable identity or storage guarantees, good for stateless services. A StatefulSet gives each pod a stable network identity and, typically, a dedicated PersistentVolume, with ordered, predictable startup and termination. I'd reach for a StatefulSet for anything where ordering or identity matters — a database cluster, Kafka brokers, anything doing leader election. For a typical API service, a Deployment is simpler and I wouldn't add StatefulSet complexity without a reason."

### Q: "How do you handle secrets management across environments?"
**Model answer:** "I avoid storing secrets in Git or bare Kubernetes Secrets, since those are only base64-encoded, not encrypted. I'd use a dedicated secrets manager — Vault, AWS Secrets Manager, or similar — with short-lived, scoped credentials pulled at runtime via an init container or CSI driver rather than baked into images or env vars at build time. Rotation should be automated on a schedule, and access should be audited per service, not shared broadly across a namespace."

### Q: "Terraform state is shared across multiple teams — how do you prevent drift and conflicts?"
**Model answer:** "Remote state with locking — S3 plus DynamoDB, or Terraform Cloud — is the baseline so two applies can't collide. Beyond that, I'd split state by service or domain rather than one monolithic state file, so a mistake in one team's plan can't blast-radius into another's. For drift, I'd run a scheduled `terraform plan` in CI against production and alert on any non-empty diff, rather than discovering drift the next time someone applies manually."

### Q: "How do you decide what to alert on vs. what just goes into a dashboard?"
**Model answer:** "I alert on symptoms that require a human to act now — SLO burn rate, error rate, saturation that threatens availability. Dashboards are for context and investigation, not paging. If an alert fires and the response is always 'that's fine, ignore it,' it shouldn't be an alert. I've had to go back and prune alerts after an on-call rotation flagged a specific one as noise — that feedback loop matters more than getting the thresholds right on day one."

### Q: "Blue/green vs. canary deployments — how do you choose?"
**Model answer:** "Blue/green gives you a fast, clean rollback since the old environment stays fully up, but it costs double the infrastructure during the cutover and doesn't limit blast radius — everyone hits the new version at once. Canary limits exposure by routing a small percentage of traffic first, which I prefer for anything with real user impact, but it needs good metrics and automated analysis to be safe, otherwise you're just guessing when to promote. For a low-traffic internal service, blue/green's simplicity usually wins; for a customer-facing API, I'd lean canary."

### Q: "Walk me through your incident response process from alert to postmortem."
**Model answer:** "Alert fires, on-call acknowledges and starts a timeline in the incident channel immediately — even before root cause is known — because reconstructing that later is unreliable. Mitigate first, understand second: rollback or scale, don't debug in prod under pressure if a safe rollback exists. Once stable, write a blameless postmortem with a clear timeline, contributing factors, and specific action items with owners and dates — not just 'we'll be more careful.' I follow up on those action items in the next sprint, because an incident that doesn't change anything will happen again."

---

## 8. Additional Interview Question Bank (Behavioral — STAR Format)

The STAR technique structures a behavioral answer into four parts: **Situation** (brief context), **Task** (what you were responsible for), **Action** (what you specifically did — the bulk of the answer), and **Result** (the outcome, ideally with a measurable or concrete detail, plus what you learned). Swap in your own real details — the structure is what matters most, and specificity is what interviewers are actually screening for (see Section 2e above).

### Q: "Tell me about a time you disagreed with a technical decision made by your team or manager."
- **Situation:** "On a past team, leadership pushed to adopt a service mesh across all services for mTLS and traffic shaping."
- **Task:** "As the engineer who'd own the operational burden, I needed to decide whether to raise an objection or go along with it."
- **Action:** "I laid out the actual cost — added failure modes, debugging complexity, and the fact that nobody on the team had deep Envoy experience — against our real traffic volume, which didn't justify it. Instead of a flat 'no,' I proposed a smaller pilot on one low-risk service first, and suggested simpler network policies and cert rotation as an alternative for the core problem we were actually trying to solve."
- **Result:** "The pilot surfaced exactly the complexity I'd flagged — config drift and unclear failure modes during a test outage — so we scaled back the full rollout and used the lighter-weight approach instead. It reinforced for me that disagreeing with data and a concrete alternative lands very differently than just pushing back on instinct."

### Q: "Describe a time you had to make a decision with incomplete information."
- **Situation:** "During an incident, we saw a spike in 5xx errors right after a deploy window, but also had a known upstream dependency that had been flaky that week."
- **Task:** "I needed to pick a mitigation fast, before I could be fully certain which of the two was the root cause."
- **Action:** "I chose to roll back the deploy first, since it was the faster and fully reversible action, rather than spending time diagnosing further while customer impact grew. I kept monitoring the upstream dependency in parallel in case the rollback didn't resolve it."
- **Result:** "The rollback fixed it — it had been the deploy. Afterward, I pushed for better health checks on that upstream dependency specifically so that if it happens again, we can rule causes in or out faster instead of guessing under pressure."

### Q: "How do you approach mentoring or supporting a more junior engineer on your team?"
- **Situation:** "I was paired with a junior engineer who'd just joined and was assigned to overhaul our alerting rules, an area with real blast radius if done wrong."
- **Task:** "My job was to help them actually own the work, not just shadow me, while making sure a mistake didn't turn into an incident."
- **Action:** "I reviewed their PRs closely early on, walked through my reasoning out loud rather than just approving or rejecting, and gradually pulled back review depth as they built track record. When they did misconfigure a threshold that caused a noisy alert, I treated it as a shared postmortem rather than a correction aimed at them."
- **Result:** "They became the de facto owner of our alerting within two months, and — more importantly — they came to me proactively the next time something looked off instead of hiding it, which told me the trust-building approach had worked."

### Q: "How do you prioritize when everything feels urgent — a production issue, a deadline, and tech debt cleanup?"
- **Situation:** "I once had an active SLO-violating production issue, a feature deadline the same week, and a known piece of tech debt that was making both harder."
- **Task:** "I had to sequence the work and communicate the tradeoff to stakeholders rather than silently dropping one of the three."
- **Action:** "I fixed the production issue immediately since it was customer-facing. Then, instead of jumping straight to the deadline work, I flagged to the team that the tech debt was actively slowing down the deadline feature, and got agreement to spend half a day addressing it first rather than treating it as a separate 'later' task."
- **Result:** "The deadline work went faster than it would have otherwise, and we hit it on time. The stakeholder conversation upfront meant nobody was surprised when the tech debt fix ate into the timeline — it was a visible, agreed tradeoff instead of a silent one."

---

## 9. Questions Worth Asking the Interviewer

- "What's your current deployment frequency, and where does it get stuck?"
- "What does your on-call rotation look like, and has it caused attrition?"
- "What does 'hands-on ownership' actually mean for this role day to day?"
- "What's the most recent incident the team had, and what changed afterward?"

These signal that you're evaluating the role seriously, not just trying to get an offer.
