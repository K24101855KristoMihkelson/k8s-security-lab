# Kubernetes Multi-Layer Security Lab (K3s, Falco eBPF & Zero-Trust NetworkPolicy)

A practical implementation of enterprise Kubernetes runtime security, kernel-level threat detection with eBPF, and declarative Zero-Trust network segmentation deployed in an isolated virtualized infrastructure.

---

## 🏗️ Architecture & Security Controls

```text
[ pfSense Firewall / NAT ]
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│ K3s Cluster (Namespace: production)                         │
│                                                             │
│   ┌─────────────────┐       TCP/6379      ┌──────────────┐  │
│   │   backend-api   │ ──────────────────> │   secure-db  │  │
│   │  (Nginx Alpine) │   (Allowed Ingress) │(Redis Alpine)│  │
│   └─────────────────┘                     └──────────────┘  │
│            ▲                                     ▲          │
│            │ (Kernel-level Syscall Monitoring)   │          │
│            └──────────────────┬──────────────────┘          │
│                               │                             │
│                  ┌─────────────────────────┐                │
│                  │   Falco (modern-ebpf)   │                │
│                  └─────────────────────────┘                │
│                               │                             │
│                   Blocked Unauthorized Traffic              │
│                     (Zero-Trust NetworkPolicy)              │
└─────────────────────────────────────────────────────────────┘
```
### Key Security Layers:
#### Perimeter Defense: Isolated virtual network segment behind a pfSense router/firewall with custom DNS and gateway routing.

#### Runtime Security (eBPF): Real-time kernel syscall tracing using Falco (modern_ebpf driver) to monitor suspicious container activity without modifying binaries.

#### Zero-Trust Network Segmentation: Granular Kubernetes NetworkPolicy restricting database ingress exclusively to authenticated microservices.

### 📁 Repository Structure
#### .
#### ├── README.md
#### └── manifests/
####    ├── app-stack.yaml     # Production namespace, Backend API, Redis & ClusterIP Service
####    └── db-policy.yaml     # Zero-Trust Ingress NetworkPolicy for Database

### 🚀 Deployment Instructions

#### 1. Deploy the Application Stack
```
kubectl apply -f manifests/app-stack.yaml
```
#### 2. Apply Zero-Trust Network Policy
```
kubectl apply -f manifests/db-policy.yaml
```
#### 3. Deploy Falco with modern eBPF probe (via Helm)
```
helm repo add falcosecurity [https://falcosecurity.github.io/charts](https://falcosecurity.github.io/charts)
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=modern_ebpf \
  --set falcoctl.enabled=false \
  --set tty=true
```
### 🔍 Threat Simulation & Validation
#### 1. Runtime Threat Detection (Falco eBPF)
##### Sensitive File Read Alert (/etc/shadow):
```
kubectl exec -it -n production <backend-pod-name> -- cat /etc/shadow
```
##### Falco Output:
```
Plaintext
16:02:26 Warning Sensitive file opened for reading by non-trusted program | file=/etc/shadow user=root process=cat container_name=nginx k8s_pod_name=backend-api-... k8s_ns_name=production
```
##### Interactive Shell Spawn Detection:
```
Bash
kubectl exec -it -n production <backend-pod-name> -- sh -c "whoami && uname -a"
```
##### Falco Output:
```
Plaintext
16:02:43 Notice A shell was spawned in a container with an attached terminal | process
```
#### 2. Zero-Trust Network Isolation (NetworkPolicy)
   ##### Legitimate Communication (Backend $\rightarrow$ Redis DB):
```
   kubectl exec -it -n production <backend-pod-name> -- nc -zvw3 secure-database 6379
   # Result: secure-database (10.43.103.108:6379) open
```
   ##### Unauthorized Lateral Movement Attempt (rogue-pod $\rightarrow$ Redis DB):
```
   kubectl run rogue-pod --image=busybox -n production --restart=Never -- sleep 3600
   kubectl exec -it -n production rogue-pod -- nc -zvw3 secure-database 6379
    Result: command terminated with exit code 1 (Dropped by NetworkPolicy)
 ```
#### 🛠️ Technologies Used
##### Orchestration: Kubernetes / K3s

##### Runtime Threat Engine: Falco (modern-eBPF)

##### Networking & Firewall: pfSense, K3s NetworkPolicy Controller (kube-router)

##### Containers: Alpine Linux, Nginx, Redis

##### Package Management: Helm v3

---

### **How to quickly create this file in the terminal:**

Käivita oma Ubuntu terminalis kaustas `~/k8s-security-lab`:

```
cat << 'EOF' > README.md
# Kubernetes Multi-Layer Security Lab (K3s, Falco eBPF & Zero-Trust NetworkPolicy)

A practical implementation of enterprise Kubernetes runtime security, kernel-level threat detection with eBPF, and declarative Zero-Trust network segmentation deployed in an isolated virtualized infrastructure.

---

## 🏗️ Architecture & Security Controls

```text
[ pfSense Firewall / NAT ]
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│ K3s Cluster (Namespace: production)                         │
│                                                             │
│   ┌─────────────────┐       TCP/6379      ┌──────────────┐  │
│   │   backend-api   │ ──────────────────> │   secure-db  │  │
│   │  (Nginx Alpine) │   (Allowed Ingress) │(Redis Alpine)│  │
│   └─────────────────┘                     └──────────────┘  │
│            ▲                                     ▲          │
│            │ (Kernel-level Syscall Monitoring)   │          │
│            └──────────────────┬──────────────────┘          │
│                               │                             │
│                  ┌─────────────────────────┐                │
│                  │   Falco (modern-ebpf)   │                │
│                  └─────────────────────────┘                │
│                               │                             │
│                   Blocked Unauthorized Traffic              │
│                     (Zero-Trust NetworkPolicy)              │
└─────────────────────────────────────────────────────────────┘
```
### Key Security Layers:
#### Perimeter Defense: Isolated virtual network segment behind a pfSense router/firewall with custom DNS and gateway routing.

#### Runtime Security (eBPF): Real-time kernel syscall tracing using Falco (modern_ebpf driver) to monitor suspicious container activity without modifying binaries.

#### Zero-Trust Network Segmentation: Granular Kubernetes NetworkPolicy restricting database ingress exclusively to authenticated microservices.

### 📁 Repository Structure
```
.
├── README.md
└── manifests/
    ├── app-stack.yaml     # Production namespace, Backend API, Redis & ClusterIP Service
```
### 🚀 Deployment Instructions
#### 1. Deploy the Application Stack
```
kubectl apply -f manifests/app-stack.yaml
```
#### 2. Apply Zero-Trust Network Policy
```
kubectl apply -f manifests/db-policy.yaml
```
#### 3. Deploy Falco with modern eBPF probe (via Helm)
```
helm repo add falcosecurity [https://falcosecurity.github.io/charts](https://falcosecurity.github.io/charts)
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=modern_ebpf \
  --set falcoctl.enabled=false \
  --set tty=true
```
### 🔍 Threat Simulation & Validation
#### 1. Runtime Threat Detection (Falco eBPF)
#### Sensitive File Read Alert (/etc/shadow):\
```
kubectl exec -it -n production <backend-pod-name> -- cat /etc/shadow
```
#### Falco Output:
```
16:02:26 Warning Sensitive file opened for reading by non-trusted program | file=/etc/shadow user=root process=cat container_name=nginx k8s_pod_name=backend-api-... k8s_ns_name=production
```
#### Interactive Shell Spawn Detection:
```
kubectl exec -it -n production <backend-pod-name> -- sh -c "whoami && uname -a"
```
#### Falco Output:
```
16:02:43 Notice A shell was spawned in a container with an attached terminal | process=sh exe_flags=EXE_WRITABLE container_name=nginx k8s_pod_name=backend-api-...
```
### 2. Zero-Trust Network Isolation (NetworkPolicy)
#### Legitimate Communication (Backend -> Redis DB):
```
kubectl exec -it -n production <backend-pod-name> -- nc -zvw3 secure-database 6379
# Result: secure-database (10.43.103.108:6379) open
```
#### Unauthorized Lateral Movement Attempt (rogue-pod -> Redis DB):
```
kubectl run rogue-pod --image=busybox -n production --restart=Never -- sleep 3600
kubectl exec -it -n production rogue-pod -- nc -zvw3 secure-database 6379
# Result: command terminated with exit code 1 (Dropped by NetworkPolicy)
```
### 🛠️ Technologies Used
#### Orchestration: Kubernetes / K3s

#### Runtime Threat Engine: Falco (modern-eBPF)

#### Networking & Firewall: pfSense, K3s NetworkPolicy Controller (kube-router)

#### Containers: Alpine Linux, Nginx, Redis

#### Package Management: Helm v3
EOF
