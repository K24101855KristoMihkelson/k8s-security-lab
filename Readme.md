# 🔒 Multi-Layer Kubernetes (K3s) Security & Observability Lab

A practical enterprise-grade security and monitoring architecture built on a lightweight **K3s** Kubernetes cluster inside an isolated network environment.

---

## 🏗️ Architecture Overview

The lab implements defense-in-depth principles across the network, cluster runtime, and monitoring layers:

1. **Perimeter Defense:** Network segmentation and firewalling via **pfSense**.
2. **Cluster & Workloads:** **K3s** orchestrating containerized services across dedicated namespaces (`production`, `falco`, `monitoring`).
3. **Zero-Trust Network Policies:** Declarative Kubernetes `NetworkPolicy` rules enforcing strict isolation.
4. **Runtime Threat Detection:** **Falco** utilizing the `modern-ebpf` probe to capture real-time Linux kernel system calls.
5. **Observability Stack (PLG + Prometheus):** Full metrics and log collection using **Prometheus**, **Grafana**, **Loki**, and **Promtail**.

```
                       +----------------------+
                       |   pfSense Firewall   |
                       +----------+-----------+
                                  |
                                  v
+-------------------------------------------------------------------------------+
|                             K3S KUBERNETES CLUSTER                            |
|                                                                               |
|  [ production ]               [ falco ]                [ monitoring ]         |
|  +---------------------+      +---------------------+  +--------------------+ |
|  | backend-api (Nginx) |      | Falco DaemonSet     |  | Prometheus         | |
|  +----------+----------+      | (modern-ebpf driver)|  | Grafana (UI:30080) | |
|             | (TCP:6379)      +----------+----------+  | Loki-Stack         | |
|             v                            |             | Promtail DaemonSet | |
|  +---------------------+                 |             +---------+----------+ |
|  | secure-database     |<----------------+                       ^            |
|  | (NetworkPolicy: OK) |  Syscall / Runtime Security             |            |
|  +---------------------+  Threat Telemetry                       |            |
|             ^                                                    |            |
|             x (DROPPED)                                          |            |
|  +---------------------+                                         |            |
|  | rogue / test pod    |                                         |            |
|  +---------------------+                                         |            |
|                                                                  |            |
|  * Nodes & Pods Metrics / Syslog Pipelines ----------------------+            |
+-------------------------------------------------------------------------------+
```
---

## 🛡️ Security Implementation & Verification

### 1. Zero-Trust Network Policy Isolation
* Configured default-deny ingress rules on `secure-database` (Redis).
* Permitted only explicit TCP traffic on port `6379` originating from `app: backend-api`.
* **Validation:** Verified using ephemeral test pods (`busybox`). Traffic from authorized backend succeeded (`open`), while rogue requests timed out and dropped.

### 2. Kernel-Level Runtime Detection (eBPF + Falco)
* Deployed Falco as a DaemonSet using the modern eBPF probe to intercept Linux syscalls directly from the kernel.
* **Detection Events:** Successfully intercepted unauthorized executions and sensitive file access attempts (e.g. `cat /etc/shadow` inside containers).

---

## 📊 Observability & Log Aggregation

* **Prometheus & Node Exporter:** Real-time hardware and cluster-wide compute resource tracking (CPU, memory, networking).
* **Grafana Dashboards:** Visualized CPU quota usage across namespaces (`falco`, `kube-system`, `monitoring`, `production`).
* **Loki + Promtail:** Centralized log pipeline ingesting Falco security events directly into Grafana Explore.

---

## 🚀 How to Reproduce

### Prerequisites
* Linux VM / Bare-metal node (Ubuntu 22.04 / 24.04 LTS)
* K3s (`v1.28+`)
* Helm (`v3+`)

### Deployment Steps
#### 1. Install K3s lightweight cluster
```
curl -sfL [https://get.k3s.io](https://get.k3s.io) | sh -
```

#### 2. Deploy Falco with modern-ebpf probe
```
helm repo add falcosecurity [https://falcosecurity.github.io/charts](https://falcosecurity.github.io/charts)
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf
```

#### 3. Deploy Zero-Trust Production Workloads
```
kubectl apply -f k8s/production-workloads.yaml
kubectl apply -f k8s/network-policy.yaml
```

#### 4. Deploy Observability Stack (Prometheus + Grafana + Loki)
```
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo add grafana [https://grafana.github.io/helm-charts](https://grafana.github.io/helm-charts)
helm update


helm install monitoring-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set alertmanager.enabled=false \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30080

helm install loki-stack grafana/loki-stack \
  --namespace monitoring \
  --set loki.persistence.enabled=false \
  --set promtail.enabled=true
```

### 📸 Screenshots & Validation Evidence
#### Cluster Status: All pods healthy across all operational namespaces.
<img width="1676" height="526" alt="image" src="https://github.com/user-attachments/assets/6913513d-8a32-49b8-84b4-76bb938cb3f5" />



#### Security Validation: Dropped rogue packets via NetworkPolicy and Falco detection logs.
<img width="1697" height="661" alt="image" src="https://github.com/user-attachments/assets/2609c1dc-a0fa-4cf4-833f-053514b6a7a9" />


#### Grafana Dashboard: Live resource monitoring and centralized log stream.
<img width="1024" height="420" alt="image" src="https://github.com/user-attachments/assets/6002b83f-2af4-410e-85cd-59b2e92957c0" />


