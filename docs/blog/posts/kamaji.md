---
date: 2025-03-11
categories:
    - Home lab
tags:
    - Kubernetes
    - Proxmox
---

# Building a Resilient Hybrid Kubernetes Cluster: Cloud Control Plane and On-Prem Workers

In today’s dynamic infrastructure landscape, balancing scalability, cost-efficiency, and security is paramount. A hybrid Kubernetes cluster—combining a managed cloud control plane with on-premises worker nodes—offers the best of both worlds. This guide walks you through my journey of creating a fault-tolerant homelab Kubernetes cluster using **Kamaji** to host the control plane in Oracle Container Engine (OKE) and worker nodes on my Proxmox homelab.

---

## **Why Go Hybrid?**

Before diving into the technical steps, let’s unpack the advantages of this architecture:

- **Fault Tolerance**: Distribute workloads across cloud and on-premises nodes to mitigate single-point failures.
- **Disaster Recovery**: Rapidly reprovision on-prem nodes using templates while relying on the cloud’s resilient control plane.
- **Security & Compliance**: Keep sensitive data on-premises with strict access controls while leveraging the cloud’s managed security for the control plane.
- **Cost Optimization**: Reduce cloud spend by running worker nodes locally while avoiding the complexity of self-hosting the control plane.

---

## **Step 1: Deploying Kamaji for Control Plane Management**

Kamaji is a Kubernetes operator that decouples the control plane from worker nodes, allowing you to host lightweight, multi-tenant control planes in a centralized management cluster. Here’s how to set it up.

### **Prerequisites**

- A functional Kubernetes cluster (e.g., OKE, EKS, or a local cluster) with a default `StorageClass`.
- **Helm**, **kubectl**, and **kubeadm** installed on your workstation.

---

### **Install Cert-Manager**

Kamaji requires Cert-Manager for TLS certificate management. Install it using Helm:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

---

### **Install Kamaji**

1. Add the Clastix Helm repository:
   ```bash
   helm repo add clastix https://clastix.github.io/charts
   ```

2. Clone the Kamaji repository and deploy:
   ```bash
   git clone https://github.com/clastix/kamaji
   cd kamaji
   helm dependency build charts/kamaji
   helm install kamaji charts/kamaji \
     -n kamaji-system \
     --create-namespace \
     --set image.tag=latest  # Use a specific tag for production
   ```

3. Verify the installation:
   ```bash
   helm status kamaji -n kamaji-system
   ```

---

### **(Optional) Install the Kamaji Console**

For a visual management interface, deploy the Kamaji Console. First, create a `console-secret.yaml` file:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kamaji-console
  namespace: kamaji-system
type: Opaque
data:
  ADMIN_EMAIL: <base64-encoded-email>      # e.g., echo "admin@example.com" | base64
  ADMIN_PASSWORD: <base64-encoded-password>
  JWT_SECRET: <base64-encoded-random-string>  # Generate with openssl rand -hex 32
  NEXTAUTH_URL: <base64-encoded-url>       # e.g., https://kamaji-console.example.com/ui
```

Apply the secret and deploy the console:

```bash
kubectl apply -f console-secret.yaml
helm -n kamaji-system install console clastix/kamaji-console -f console-values.yaml
```

Access the console via port-forwarding or an ingress controller using the credentials from the secret.

---

## **Step 2: Deploying a Tenant Control Plane**

With Kamaji operational, create a tenant control plane (TCP) to manage your workload cluster.

1. Apply a TCP manifest (e.g. [tcp-template.yaml](https://raw.githubusercontent.com/zazathomas/Homelab/refs/heads/main/K8s/kamaji/tcp-template.yaml)):
   ```bash
   export TENANT_NAMESPACE=<specify namespace control plane is deployed>
   export TENANT_NAME=<specify control plane name>
   kubectl apply -f tcp-template.yaml -n ${TENANT_NAMESPACE}
   ```

2. Retrieve the kubeconfig for your tenant cluster:
   ```bash
   kubectl get secret -n ${TENANT_NAMESPACE} ${TENANT_NAME}-admin-kubeconfig -o jsonpath='{.data.admin\.conf}' | base64 -d > homelab-tcp.kubeconfig
   ```

---

## **Step 3: Provisioning On-Premises Worker Nodes on Proxmox**

### **Automate Node Bootstrap with Yaki**

[Yaki](https://github.com/clastix/yaki) simplifies Kubernetes node provisioning. Use my customized yaki script to prepare Ubuntu 22.04/24.04 VMs:

```bash
# Install dependencies and bootstrap the node
sudo apt install conntrack socat -y
curl -sfL https://raw.githubusercontent.com/zazathomas/Homelab/main/K8s/kamaji/yaki.sh > yaki.sh && chmod +x yaki.sh
sudo KUBERNETES_VERSION=v1.31.4 ./yaki.sh bootstrap
rm yaki.sh
```

**Pro Tip**: After configuring a VM, convert it to a Proxmox template for rapid cloning.

---

## **Step 4: Joining Workers to the Cluster**

1. **Retrieve the Join Command**:
   ```bash
   JOIN_CMD=$(kubeadm --kubeconfig=homelab-tcp.kubeconfig token create --print-join-command)
   ```

2. **Execute the Command on Each Worker**:
   ```bash
   WORKERS=("192.168.0.231" "192.168..0.232" "192.168..0.233") # Replace this with your worker IPs
   for WORKER_IP in "${WORKERS[@]}"; do
     ssh user@$WORKER_IP -t "sudo $JOIN_CMD"
   done
   ```

---

## **Step 5: Installing Core Cluster Components**

After workers join, deploy these essentials:

1. **CNI Plugin** (e.g., Cilium):
   ```bash
   helm repo add cilium https://helm.cilium.io/
   helm install cilium cilium/cilium --version 1.17.1 --namespace kube-system
   ```

2. **Storage Provider** (e.g. Longhorn):
   ```bash
   helm repo add longhorn https://charts.longhorn.io
   helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.8.1
   ```

3. **Metrics Server**:
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

---

## **Final Steps: Validation**

Verify cluster health:

```bash
kubectl --kubeconfig=homelab-tcp.kubeconfig get nodes
```

Ensure all nodes show `Ready` status, and deploy a test workload:

```bash
kubectl --kubeconfig=homelab-tcp.kubeconfig run nginx --image=nginx --port=80
```

---

## **Conclusion**

You’ve now built a hybrid Kubernetes cluster that merges the reliability of a managed cloud control plane with the flexibility of on-premises workers. This setup is ideal for homelabs or edge computing scenarios where data sovereignty and cost control are priorities.

**Next Steps**:
- Explore Kamaji’s multi-tenancy features for isolating workloads.
- Implement a CI/CD pipeline to deploy applications across hybrid nodes.
- Set up monitoring with Prometheus and Grafana.

By embracing hybrid Kubernetes, you’re future-proofing your infrastructure—one cluster at a time. 🚀
