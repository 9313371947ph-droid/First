# Section 03: Setting Up Your Lab Environment

## 🎯 Learning Objectives
- Choose the right Kubernetes setup for your needs
- Install Minikube for single-node clusters
- Install Kind for multi-node clusters
- Install kubectl CLI tool
- Deploy your first cluster and verify it works

## Choose Your Setup

### Option Comparison

| Tool | Best For | Nodes | Resource Usage | Persistence |
|------|----------|-------|----------------|-------------|
| **Minikube** | Learning, single-node testing | 1 (configurable) | Medium | ✅ Yes |
| **Kind** | CI/CD, multi-node testing | 1+ | Low | ❌ No (ephemeral) |
| **k3d** | Lightweight, edge computing | 1+ | Very Low | ❌ No |
| **Docker Desktop** | Mac/Windows beginners | 1 | High | ✅ Yes |
| **Managed K8s** | Production (EKS, GKE, AKS) | 1+ | Pay-per-use | ✅ Yes |
| **kubeadm** | Learning production setup | 1+ | High | ✅ Yes |

### Recommendation for This Course

**For Beginners:** Start with **Minikube**
- Easy setup
- Persistent cluster
- Good documentation
- Supports addons (dashboard, metrics, etc.)

**For Advanced Users:** Use **Kind**
- Fast cluster creation
- Multi-node support
- Great for CI/CD testing
- Closer to production behavior

---

## Prerequisites

Before you begin, ensure you have:

1. **Virtualization enabled** in BIOS (for Minikube)
2. **Docker installed** (all methods use Docker)
3. **At least 4GB RAM free** (8GB recommended)
4. **20GB free disk space**
5. **kubectl** (we'll install this)

### Check Virtualization (Linux)
```bash
# Check if virtualization is enabled
egrep -c '(vmx|svm)' /proc/cpuinfo

# 0 = disabled, >0 = enabled
```

### Install Docker (if not installed)
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker run hello-world
```

---

## Step 1: Install kubectl (Kubernetes CLI)

### Linux (AMD64)
```bash
# Download latest release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make executable
chmod +x kubectl

# Move to PATH
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

### macOS
```bash
# Using Homebrew
brew install kubectl

# Or download directly
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Windows (PowerShell)
```powershell
# Download
curl.exe -LO "https://dl.k8s.io/release/$(curl.exe -L -s https://dl.k8s.io/release/stable.txt)/bin/windows/amd64/kubectl.exe"

# Move to a folder in your PATH
```

### Verify kubectl Installation
```bash
kubectl version --client --output=yaml
```

Expected output:
```yaml
clientVersion:
  gitVersion: v1.29.0
  major: "1"
  minor: "29"
```

---

## Step 2A: Install Minikube (Recommended for Beginners)

### Linux
```bash
# Download latest Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Verify
minikube version
```

### macOS
```bash
# Using Homebrew
brew install minikube

# Or download directly
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

### Windows
```powershell
# Using Chocolatey
choco install minikube

# Or download installer from:
# https://minikube.sigs.k8s.io/docs/start/
```

---

## Step 2B: Install Kind (Alternative)

### Linux
```bash
# Download
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

# Make executable
chmod +x ./kind

# Move to PATH
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version
```

### macOS
```bash
# Using Homebrew
brew install kind

# Or download directly
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-darwin-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

---

## Step 3: Create Your First Cluster

### Option A: Minikube Cluster

#### Basic Single-Node Cluster
```bash
# Start Minikube with default settings
minikube start

# Start with specific resources (recommended)
minikube start --cpus=2 --memory=4096 --disk-size=20gb

# Start with Kubernetes version
minikube start --kubernetes-version=v1.29.0

# Start with multiple nodes
minikube start --nodes=3
```

#### Verify Cluster
```bash
# Check cluster status
minikube status

# Expected output:
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Generated

# Check kubectl context
kubectl config current-context
# Should show: minikube

# View cluster info
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://192.168.49.2:8443
# CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

#### Access Kubernetes Dashboard
```bash
# Enable dashboard addon
minikube addons enable dashboard

# Open dashboard in browser
minikube dashboard
```

#### Useful Minikube Commands
```bash
# Pause Kubernetes (saves resources)
minikube pause

# Unpause
minikube unpause

# Stop cluster (keeps data)
minikube stop

# Delete cluster (removes everything)
minikube delete

# List profiles (multiple clusters)
minikube profile list

# SSH into node
minikube ssh

# Get IP address
minikube ip
```

---

### Option B: Kind Cluster

#### Basic Single-Node Cluster
```bash
# Create cluster
kind create cluster

# Create with specific name
kind create cluster --name my-cluster

# Create with specific Kubernetes version
kind create cluster --image kindest/node:v1.29.0
```

#### Multi-Node Cluster
Create a config file `kind-config.yaml`:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

```bash
# Create multi-node cluster
kind create cluster --config kind-config.yaml --name multi-node
```

#### Verify Cluster
```bash
# List clusters
kind get clusters

# Get kubeconfig context
kubectl config current-context
# Should show: kind-multi-node

# View nodes
kubectl get nodes

# Expected output:
# NAME                  STATUS   ROLES           AGE   VERSION
# multi-node-control-plane   Ready    control-plane   2m    v1.29.0
# multi-node-worker          Ready    <none>          2m    v1.29.0
# multi-node-worker2         Ready    <none>          2m    v1.29.0
# multi-node-worker3         Ready    <none>          2m    v1.29.0
```

#### Useful Kind Commands
```bash
# List clusters
kind get clusters

# Delete cluster
kind delete cluster --name my-cluster

# Delete all clusters
kind delete clusters --all

# Export kubeconfig
kind export kubeconfig --name my-cluster

# Load Docker image into cluster
kind load docker-image my-image:latest --name my-cluster

# Get cluster nodes
kind get nodes --name my-cluster
```

---

## Step 4: Verify Everything Works

### Test 1: Check Nodes
```bash
kubectl get nodes

# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   5m    v1.29.0
```

### Test 2: Check System Pods
```bash
kubectl get pods -n kube-system

# Should see running pods like:
# coredns-xxxxx          Running
# etcd-minikube          Running
# kube-apiserver-xxx     Running
# kube-controller-xxx    Running
# kube-proxy-xxxx        Running
# kube-scheduler-xxx     Running
# storage-provisioner    Running
```

### Test 3: Deploy Your First Application
```bash
# Deploy nginx
kubectl create deployment nginx --image=nginx:latest

# Check deployment
kubectl get deployments

# Scale to 3 replicas
kubectl scale deployment nginx --replicas=3

# Check pods
kubectl get pods

# Expose as service
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Check service
kubectl get services

# In Minikube, access via:
minikube service nginx --url
```

### Test 4: Run a Simple Pod
```bash
# Create a simple pod
kubectl run test-pod --image=busybox --restart=Never -- sleep 3600

# Check pod
kubectl get pods

# Execute command inside pod
kubectl exec test-pod -- echo "Hello Kubernetes!"

# Expected output: Hello Kubernetes!

# Delete pod
kubectl delete pod test-pod
```

### Test 5: Check Cluster Resources
```bash
# View resource usage (requires metrics-server)
kubectl top nodes

# If not available, enable in Minikube:
minikube addons enable metrics-server
kubectl top nodes
```

---

## Step 5: Configure kubectl Autocompletion

### Bash (Linux/macOS)
```bash
# Add to ~/.bashrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# Test: Type 'kubectl get [TAB]'
```

### Zsh (macOS/Linux)
```bash
# Add to ~/.zshrc
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
source ~/.zshrc

# If using oh-my-zsh, also add:
echo 'compdef __start_kubectl kubectl' >> ~/.zshrc
```

### PowerShell (Windows)
```powershell
# Add to profile
kubectl completion powershell | Out-String | Invoke-Expression
```

---

## Troubleshooting Common Issues

### Issue 1: Minikube Won't Start
```bash
# Check virtualization
egrep -c '(vmx|svm)' /proc/cpuinfo

# Delete and recreate
minikube delete
minikube start --driver=docker

# Try different driver
minikube start --driver=virtualbox
```

### Issue 2: Permission Denied
```bash
# Fix Docker permissions
sudo usermod -aG docker $USER
newgrp docker

# Or run with sudo (not recommended)
sudo minikube start
```

### Issue 3: Insufficient Resources
```bash
# Stop other VMs/containers
docker ps
docker stop <container-id>

# Start with fewer resources
minikube start --cpus=1 --memory=2048

# Check available memory
free -h
```

### Issue 4: kubectl Can't Connect
```bash
# Check context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch to correct context
kubectl config use-context minikube

# Or regenerate
minikube update-context
```

### Issue 5: Kind Cluster Creation Fails
```bash
# Check Docker is running
docker ps

# Check Docker socket permissions
ls -la /var/run/docker.sock

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Delete failed cluster
kind delete cluster --name my-cluster

# Recreate with verbose output
kind create cluster -v5
```

---

## Cleanup (When Done)

### Minikube
```bash
# Stop (preserves data)
minikube stop

# Delete (removes everything)
minikube delete

# Delete specific profile
minikube delete -p my-cluster
```

### Kind
```bash
# Delete specific cluster
kind delete cluster --name my-cluster

# Delete all clusters
kind delete clusters --all
```

### Remove Tools (Optional)
```bash
# Remove kubectl
sudo rm /usr/local/bin/kubectl

# Remove Minikube
sudo rm /usr/local/bin/minikube
rm -rf ~/.minikube

# Remove Kind
sudo rm /usr/local/bin/kind
```

---

## Quick Reference Card

### Minikube Essentials
```bash
minikube start                    # Start cluster
minikube stop                     # Stop (preserve)
minikube delete                   # Delete all
minikube status                   # Check status
minikube dashboard                # Open UI
minikube ssh                      # SSH into node
minikube ip                       # Get IP
minikube service <name>           # Access service
```

### Kind Essentials
```bash
kind create cluster               # Create cluster
kind get clusters                 # List clusters
kind delete cluster               # Delete cluster
kind load docker-image <img>      # Load image
kind export kubeconfig            # Update kubeconfig
```

### kubectl Essentials
```bash
kubectl get nodes                 # List nodes
kubectl get pods                  # List pods
kubectl get services              # List services
kubectl get all                   # List everything
kubectl describe <resource> <name># Detailed info
kubectl logs <pod-name>           # View logs
kubectl exec -it <pod> -- bash    # Shell into pod
kubectl delete <resource> <name>  # Delete resource
kubectl apply -f file.yaml        # Apply config
```

---

## 🧠 Knowledge Check

**Q1:** What's the main difference between Minikube and Kind?

**Q2:** Which component stores the Kubernetes cluster state?

**Q3:** How do you access a service running in Minikube from your browser?

**Q4:** What command shows all pods in all namespaces?

**Q5:** Why might `kubectl top nodes` not work initially?

---

**Answers:**
<details>
<summary>Click to reveal answers</summary>

**A1:** Minikube creates a persistent single-node cluster ideal for learning, while Kind creates ephemeral clusters using Docker containers, better for CI/CD and multi-node testing.

**A2:** etcd stores the cluster state. In Minikube/kind, it runs as a container/pod.

**A3:** Use `minikube service <service-name>` which opens the browser or provides the URL.

**A4:** `kubectl get pods --all-namespaces` or `kubectl get pods -A`

**A5:** The metrics-server addon is not installed by default. Enable it with `minikube addons enable metrics-server`.

</details>

## 🚀 What's Next?

Now that your lab is ready, let's learn about the fundamental building block of Kubernetes:

👉 **Next:** [Section 04: Pods - The Atomic Unit](./04-pods.md)

---

## Appendix: Alternative Setups

### Docker Desktop Kubernetes
```bash
# Enable in Docker Desktop settings
# Preferences → Kubernetes → Enable Kubernetes

# Verify
kubectl get nodes
```

### k3d (Lightweight)
```bash
# Install
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Create cluster
k3d cluster create mycluster

# Delete
k3d cluster delete mycluster
```

### Cloud Managed Kubernetes
```bash
# AWS EKS
aws eks update-kubeconfig --name my-cluster

# Google GKE
gcloud container clusters get-credentials my-cluster

# Azure AKS
az aks get-credentials --resource-group myRG --name my-cluster
```
