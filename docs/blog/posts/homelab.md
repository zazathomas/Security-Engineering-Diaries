---
date: 2025-03-11
categories:
    - Homelab
tags:
    - Kubernetes
    - Proxmox
---

**Title: Building My Homelab: A Security Engineer’s Journey into Cloud-Native Mastery**

As a security engineer specializing in cloud-native environments, DevSecOps, and Kubernetes, my homelab isn’t just a hobby—it’s a mission-critical sandbox where theory meets practice, and vulnerabilities meet solutions. Over the years, this lab has evolved from a humble single board mini PC to a sprawling hybrid ecosystem spanning my home *and* Oracle Cloud. Here’s why I built it, what I’ve learned, and how it bridges my professional expertise with hands-on experimentation.

---

### **The Catalyst: Why a Homelab?**

In cloud-native security, textbook knowledge only gets you so far. Real threats emerge in the gaps between design and deployment, and defending modern infrastructure demands fluency in the tools orchestrating it. My homelab became the proving ground where I could:
- **Simulate real-world attacks** on containerized workloads.
- **Stress-test security controls** like runtime monitoring, network policies, and CI/CD guardrails.
- **Master emerging tech** (e.g., Cilium’s Gateway API, eBPF-powered security) without risking production systems.
- **Automate my home** while hardening IoT devices against lateral movement.

But beyond upskilling, the lab taught me resilience. When a misconfigured firewall once locked me out of my managed OKE cluster, or a Kubernetes upgrade broke my entire ingress stack, those "disasters" became my best teachers.

---

### **The Architecture: Hybrid, Scalable, and Unapologetically Over-Engineered**

My homelab is a blend of on-premises hardware and cloud resources, optimized for learning and redundancy:

#### **1. Virtualization with Proxmox**
- **Why Proxmox?** It’s open-source, supports nested virtualization, and lets me mimic multi-tenant environments. I run clusters of lightweight VMs (Ubuntu/Debian) for Kubernetes nodes and isolated "sacrificial" labs for malware analysis.
- **Key Learnings**:
  - Resource quotas prevent a misbehaving VM from starving others (critical for shared hardware).
  - Snapshots are a double-edged sword—great for rollbacks, but reliance on them can mask flawed automation.

#### **2. Containerization & Kubernetes**
- **Docker**: My gateway to containers. I containerized everything—from legacy apps to home automation services—to practice vulnerability scanning and least-privilege principles.
- **Kubernetes**: The crown jewel. My hybrid cluster combines:
  - **On-prem nodes**: Virtual kubernetes workers on my proxmox cluster.
  - **Oracle Cloud Always-Free Tier**: A cloud-based kamaji control plane spanning 2 availability zones.
  - **Security Tools**:
    - **Kyverno** for policy-as-code (e.g., blocking privileged pods).
    - **Falco** for runtime threat detection.
    - **NeuVector** for zero-trust container security, enforcing network policies and continuouds monitoring for kuberneetes clusters.
    - **Cilium** for network policies and replacing legacy ingress with **Cilium Gateway API** (a game-changer for L7 routing).

#### **3. Observability & Incident Response**
- **Prometheus** and **Grafana**: The backbone of my monitoring stack. Prometheus scrapes metrics from Kubernetes, Proxmox, and even IoT devices, while Grafana dashboards visualize everything from node resource usage to API latency.
- **Wazuh**: My SIEM (Security Information and Event Management) tool, aggregating logs from Kubernetes, Docker, and network devices. It triggers alerts for suspicious activity, like unexpected privilege escalation in pods.
- **n8n**: Repurposed as a lightweight SOAR (Security Orchestration, Automation, and Response) platform. It automates responses to common alerts—e.g., quarantining a compromised VM via Proxmox’s API or blocking an IP in AdGuard Home.

#### **4. Identity & Access Management**
- **Keycloak**: My centralized identity provider. It handles authentication for ArgoCD, Grafana, and even my media server stack. Integrating it with Kubernetes via OAuth2 Proxy taught me the nuances of securing SSO in distributed systems.

#### **5. CI/CD & GitOps**
- **ArgoCD**: The GitOps engine driving my deployments. Every change to my GitHub repo (infrastructure-as-code, app manifests) is automatically synced to the cluster. Rollbacks are as simple as reverting a commit.
- **Apko**: Used to build secure, minimal container images for APIs. By distroless base images and static binaries, I’ve reduced CVE surface areas in custom apps.

#### **6. Networking & Data Flow**
- **AdGuard Home**: Blocks ads/malware at the DNS layer, with custom rules to silence chatty IoT devices. Integrated with Prometheus to log query trends.
- **Traefik & Cilium Gateway API**: Traefik handles TLS termination for public-facing apps, while Cilium Gateway API manages internal L7 routing (e.g., gRPC traffic between microservices).
- **Kafka**: Acts as the message broker for event-driven workflows. For example, IoT sensor data streams into Kafka, processed by Flink for anomaly detection, and stored in TimescaleDB.

#### **7. Home Automation & Media**
- **Home Assistant**: Runs in a Kubernetes pod, with Zigbee2MQTT and Node-RED automating lights, HVAC, and security cameras.
- **Media Server Stack**: Jellyfin for streaming, backed by RAID-Z2 storage in Proxmox. Secured with Keycloak authentication and NeuVector network policies to isolate it from other services.

---

### **The Hybrid Cloud: Spanning Home and Oracle Cloud**

To mirror enterprise environments, I extended my lab to Oracle Cloud’s Always-Free tier:
- **Control Plane Resilience**: Hosting Kubernetes control plane components in the cloud ensures availability even if my home loses power.
- **Cost Efficiency**: Free ARM compute instances handle resource-heavy tasks (e.g., CI/CD runners for Tekton pipelines).
- **Geo-Redundancy**: Critical services like VPN (WireGuard) and backup storage (MinIO) replicate across regions.

**Example Project**: A cloud-hosted Vault instance manages secrets for both environments, while SPIFFE/SPIRE enforces workload identity across clusters.

---

### **Key Learnings (and Mistakes)**

1. **DevSecOps Isn’t Just Tools—It’s Culture**:
   - Automating security checks in CI/CD (e.g., Trivy scans, Terraform policy enforcement) caught misconfigurations early.
   - But without documenting *why* a policy exists (e.g., blocking `latest` tags), teams bypass them.

2. **Containers ≠ Security**:
   - Even with rootless containers, a compromised app can exploit kernel vulnerabilities.
   - Solution: Regular `gVisor` sandboxing trials and seccomp profiles tailored to each workload.

3. **Power Outages Are the Ultimate Test**:
   - After a 12-hour blackout corrupted my local Ceph storage, I adopted a 3-2-1 backup rule:
     - **3 copies** of data (local, Oracle Cloud, and Cloudflare R2).
     - **2 formats** (raw disks and containerized volumes).
     - **1 air-gapped backup** (an offline SSD updated monthly).

4. **Documentation Is Survival**:
   - My early "I’ll remember how this works" phase led to days of reverse-engineering my own setups. Now, everything is codified in IaC (Terraform + Ansible) and Obsidian.

---

### **What’s Next?**

- **eBPF Deep Dive**: Expanding Cilium’s security audits using Hubble for network observability.
- **Zero Trust for IoT**: Implementing SPIFFE identities for smart devices to authenticate to Home Assistant.
- **Chaos Engineering**: Automating cluster failure simulations with Chaos Mesh to validate backups and policies.

---

### **Conclusion: Why This All Matters**

For security engineers, a homelab isn’t about the tech—it’s about cultivating a mindset. By breaking (and fixing) things in a controlled environment, I’ve learned to:
- Anticipate attack vectors in cloud-native tooling.
- Balance security with usability (no, not every pod needs a `securityContext` with `drop: ALL`).
- Respect the shared responsibility model: Cloud providers secure infrastructure; *we* secure workloads.

If you’re in tech, build a lab. Let it break. Learn from it. Because the next zero-day or misconfigured S3 bucket you thwart might just owe its defeat to the lessons learned in your basement at 2 a.m.

---

**Tools & Projects Mentioned**:
- **Virtualization**: Proxmox, KVM
- **Containers**: Docker, containerd, gVisor, Apko
- **Kubernetes**: K3s, Cilium, Kyverno, Falco, Tetragon, NeuVector, ArgoCD
- **Networking**: AdGuard Home, Traefik, Cilium Gateway API, Kafka
- **Monitoring**: Prometheus, Grafana, Wazuh
- **Security**: Vault, SPIFFE/SPIRE, Trivy, Keycloak
- **Automation**: n8n, Home Assistant, Node-RED
- **Media**: Jellyfin, Zigbee2MQTT

*Diagram*: [Imagine a hand-drawn sketch here of my hybrid setup—Proxmox VMs, Kubernetes clusters, and Oracle Cloud nodes all connected via Traefik, with AdGuard Home as the DNS watchdog.]

