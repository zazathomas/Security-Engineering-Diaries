---
date: 2025-03-08
categories:
    - Homelab
tags:
    - Kubernetes
    - Homelab
authors:
   - zaza
---

# **Why I Built a Homelab: A Security Engineer’s Journey into Cloud-Native Mastery**

As a security engineer specializing in cloud-native environments, DevSecOps, and Kubernetes, my homelab isn’t just a hobby—it’s a mission-critical sandbox where theory meets practice, and vulnerabilities meet solutions. Over the years, this lab has evolved from a humble single board mini PC to a sprawling hybrid ecosystem spanning my home *and* the public cloud. Here’s why I built it, what I’ve learned, and how it bridges my professional expertise with hands-on experimentation.
<!-- more -->
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
    - **Falco and Tetragon** for runtime threat detection.
    - **NeuVector** for zero-trust container security, enforcing network policies and continuous monitoring for kuberneetes clusters.
    - **Cilium** for network policies and replacing legacy ingress with **Cilium Gateway API** (a game-changer for L7 routing).

#### **3. Observability & Incident Response**
- **Prometheus** and **Grafana**: The backbone of my monitoring stack. Prometheus scrapes metrics from Kubernetes, Proxmox, and even IoT devices, while Grafana dashboards visualize everything from node resource usage to API latency.
- **Wazuh**: My SIEM (Security Information and Event Management) tool, aggregating logs from Kubernetes, Docker, and network devices. It triggers alerts for suspicious activity, like unexpected privilege escalation in pods.
- **n8n**: Repurposed as a lightweight SOAR (Security Orchestration, Automation, and Response) platform. It automates responses to common alerts—e.g., quarantining a compromised VM via Proxmox’s API or blocking an IP in AdGuard Home.

#### **4. Identity & Access Management**
- **Keycloak**: Centralized identity provider managing authentication flows for ArgoCD, Grafana, and media services. Integrated with Kubernetes through OIDC claim mapping, enforcing group-based RBAC policies across environments.

- **Teleport**: Identity-native infrastructure access platform providing:
    - **Zero-Trust Kubernetes Access**: Short-lived X.509 certificates replacing static kubeconfig files, authenticated via Keycloak OIDC integration
    - **Unified Audit Trail**: Session recording with ASN.1 BER-encoded audit logs for forensic-ready SSH/K8s access histories
    - **Just-in-Time Permissions**: Time-bound role activation through Keycloak identity assertions (e.g., temporary cluster-admin during incidents)

- **SPIFFE/SPIRE**: Workload identity federation across hybrid clusters, enabling:
    - Automated mTLS certificate issuance for service-to-service communication
    - Fine-grained attestation policies based on kernel measurements

#### **5. DevSecOps & GitOps Automation Framework**

- **ArgoCD**: Core GitOps controller enforcing security-first deployment practices:  
    - **Policy-as-Code Gates**: Integrated OPA/Gatekeeper checks validating resource constraints pre-sync  
    - **Audit-Ready Versioning**: Immutable Git commit-triggered deployments with signed Kustomize manifests  
    - **SSO-Enabled Governance**: Keycloak-integrated RBAC controlling environment promotions (dev → staging → prod)  

- **Akto API Security**: Shift-left API protection framework implementing automated OpenAPI spec validation during CI builds  

- **DefectDojo**: Unified vulnerability management platform providing:  
    - **Aggregated Risk Scoring**: Correlation of SAST/DAST/SCA findings across microservices  
    - **Compliance Mapping**: NIST 800-53 control alignment for critical vulnerabilities  
    - **Automated Remediation**: GitLab MR generation with patched dependencies via Dependabot integration  


#### **6. Data & Event Streaming**

- **Kafka Event Mesh**: Mission-critical nervous system for security operations:  
    - **Incident Forensics**: Immutable audit log retention for SOC2-compliant event replay  
    - **Workflow Choreography**: Triggering automated responses via Knative Functions (e.g., quarantine workflows)  

- **MinIO Object Storage**: S3-compatible backbone for:  
    - **Artifact Registry**: Secure storage of static files and SBOM archives  
    - **Immutable Backups**: Write-Once-Read-Many (WORM) policies for Velero cluster snapshots with cross instance replication.

- **Kopia Enterprise-Grade DR**: Cross-platform recovery solution featuring:  
    - **Hybrid Cloud Backups**: Unified policy management for local/Oracle Cloud storage targets  
    - **Military-Grade Encryption**: AES-256-GCM protected backups with Keycloak-managed keys  
    - **Ransomware Resilience**: Air-gapped backups via automated MinIO bucket isolation triggers  

#### **7. Networking & Data Flow**
- **AdGuard Home**: Blocks ads/malware at the DNS layer, with custom rules to silence chatty IoT devices. Integrated with Prometheus to log query trends.
- **Traefik & Cilium Gateway API**: Traefik handles TLS termination for public-facing apps, while Cilium Gateway API manages internal L7 routing (e.g., gRPC traffic between microservices).
- **Kafka**: Acts as the message broker for event-driven workflows. For example, IoT sensor data streams into Kafka, processed by Flink for anomaly detection, and stored in TimescaleDB.

#### **8. Home Automation & Media**
- **Home Assistant**: Runs in a Kubernetes pod, with Zigbee2MQTT and Node-RED automating lights, HVAC, and security cameras.
- **Media Server Stack**: Jellyfin for streaming, backed by RAID-Z2 storage in Proxmox. Secured with Keycloak authentication and NeuVector network policies to isolate it from other services.


---

### **Key Learnings (and Mistakes)**

1. **DevSecOps Isn’t Just Tools—It’s Culture**:
   - Automating security checks in CI/CD (e.g., Trivy scans, Terraform policy enforcement) caught misconfigurations early.
   - But without documenting *why* a policy exists (e.g., blocking `latest` tags), teams bypass them.

2. **Containers ≠ Security**:
   - Even with rootless containers, a compromised app can exploit kernel vulnerabilities.
   - Solution: Regular `gVisor` sandboxing trials and seccomp profiles tailored to each workload.

3. **Power Outages Are the Ultimate Test**:
   - After a 12-hour blackout corrupted my local storage, I adopted a 3-2-1 backup rule:
     - **3 copies** of data (local, Oracle Cloud, and Cloudflare R2).
     - **2 formats** (raw disks and containerized volumes).
     - **1 air-gapped backup** (an offline SSD updated monthly).

4. **Documentation Is Survival**:
   - My early "I’ll remember how this works" phase led to days of reverse-engineering my own setups. Now, everything is codified in IaC (Terraform + Ansible) and well documented on Obsidian.

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


