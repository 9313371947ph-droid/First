# Section 02: Kubernetes Architecture Deep Dive

## 🎯 Learning Objectives
- Understand the Kubernetes cluster architecture
- Learn the role of each control plane component
- Understand worker node components
- Know how components communicate
- Visualize the flow of a pod deployment

## Overview: The Big Picture

A Kubernetes cluster consists of two main parts:

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────┐      ┌─────────────────────────┐  │
│  │   CONTROL PLANE      │      │     WORKER NODES        │  │
│  │   (Master Node)      │      │   (Minion Nodes)        │  │
│  │                      │      │                         │  │
│  │  ┌────────────────┐  │      │  ┌───────────────────┐  │  │
│  │  │ API Server     │  │      │  │ Kubelet           │  │  │
│  │  └────────────────┘  │      │  └───────────────────┘  │  │
│  │  ┌────────────────┐  │      │  ┌───────────────────┐  │  │
│  │  │ ETCD           │  │      │  │ Kube-proxy        │  │  │
│  │  └────────────────┘  │      │  └───────────────────┘  │  │
│  │  ┌────────────────┐  │      │  ┌───────────────────┐  │  │
│  │  │ Scheduler      │  │      │  │ Containers/Pods   │  │  │
│  │  └────────────────┘  │      │  └───────────────────┘  │  │
│  │  ┌────────────────┐  │      │                         │  │
│  │  │ Controller Mgr │  │      │  Node 2, 3, 4...       │  │
│  │  └────────────────┘  │      │                         │  │
│  │                      │      │                         │  │
│  └──────────────────────┘      └─────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Part 1: Control Plane (Master Node)

The **Control Plane** makes global decisions about the cluster and detects/responds to cluster events.

### 1.1 kube-apiserver (API Server) ⭐

**Role:** The front-end of the Kubernetes control plane

**What it does:**
- Exposes the Kubernetes API (RESTful)
- Validates and configures data for API objects (pods, services, etc.)
- Acts as a hub for all component communication
- Authentication and authorization
- Admissions control (validates requests)

**Key Characteristics:**
- ✅ **Stateless**: Can be scaled horizontally
- ✅ **Only component that talks to etcd directly**
- ✅ **All communication flows through it**
- 🔒 **Security gateway**: Authenticates users and services

**How you interact with it:**
```bash
# Every kubectl command talks to API Server
kubectl get pods          # → API Server → etcd
kubectl apply -f pod.yaml # → API Server → validates → etcd
kubectl delete pod xyz    # → API Server → etcd
```

**Architecture Flow:**
```
User/Admin (kubectl)
      ↓
┌─────────────┐
│ API Server  │ ← All authentication & validation happens here
└─────────────┘
      ↓
┌─────────────┐
│    etcd     │ ← Only API Server can write here
└─────────────┘
```

**Example Request:**
```bash
# When you run:
kubectl run nginx --image=nginx

# This HTTP request goes to API Server:
POST /api/v1/namespaces/default/pods
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx"
  },
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx"
    }]
  }
}
```

---

### 1.2 etcd (Distributed Key-Value Store) 💾

**Role:** The brain/memory of Kubernetes

**What it does:**
- Stores ALL cluster data (configuration, state, metadata)
- Consistent and highly-available key-value store
- Source of truth for the entire cluster

**Key Characteristics:**
- 🗄️ **Stores everything**: Pods, Services, ConfigMaps, Secrets, Namespaces, etc.
- 🔐 **Critical security**: Access should be restricted to API Server only
- 📊 **Small data, high value**: Typically < 1GB but absolutely critical
- 🔄 **Distributed**: Runs as a cluster (usually 3 or 5 nodes) for HA

**What's stored in etcd?**
```
/kubernetes
├── registry
│   ├── pods
│   │   ├── default/nginx-pod-1
│   │   └── kube-system/coredns-xyz
│   ├── services
│   ├── deployments
│   ├── configmaps
│   ├── secrets
│   └── ...
└── ...
```

**Important Notes:**
- ⚠️ **Backup etcd regularly!** Loss = cluster loss
- ⚠️ **Slow disk = slow cluster** (needs fast SSD)
- ⚠️ **Never access directly** (always go through API Server)

**Etcd Commands (for admins):**
```bash
# Check etcd health
etcdctl endpoint health

# Backup etcd
etcdctl snapshot save backup.db

# List all keys (careful!)
etcdctl get / --prefix --keys-only
```

---

### 1.3 kube-scheduler 📅

**Role:** Assigns pods to nodes

**What it does:**
- Watches for newly created pods with no assigned node
- Selects the best node for each pod
- Considers resource requirements, constraints, policies

**Scheduling Decision Factors:**
1. **Resource Requirements** (CPU, Memory)
2. **Node Affinity/Anti-Affinity** (prefer/avoid certain nodes)
3. **Taints and Tolerations** (repel/attract pods)
4. **Pod Affinity/Anti-Affinity** (co-locate/separate pods)
5. **Data Locality** (storage proximity)
6. **Policy Constraints** (compliance, security)

**Scheduling Process:**
```
Step 1: Filtering (Which nodes CAN run this pod?)
  ├─ Node 1: Insufficient memory ❌
  ├─ Node 2: Meets all requirements ✅
  └─ Node 3: Has taint, pod has no toleration ❌

Step 2: Scoring (Which is the BEST node?)
  ├─ Node 2: Score 85 (good resources, some load)
  └─ Node 4: Score 92 (better fit, less loaded) ← WINNER!

Step 3: Bind
  └─ Pod assigned to Node 4
```

**Example with Resource Requests:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
```

Scheduler ensures the node has at least 512Mi memory and 500m CPU available.

---

### 1.4 kube-controller-manager 🎮

**Role:** Runs controller processes

**What it does:**
- Manages various controllers that regulate cluster state
- Each controller handles a specific aspect of cluster management

**Key Controllers:**

| Controller | Responsibility |
|-----------|----------------|
| **ReplicaSet Controller** | Maintains correct number of pod replicas |
| **Deployment Controller** | Manages deployments and rolling updates |
| **Service Controller** | Creates cloud provider load balancers |
| **Node Controller** | Monitors node health, handles failures |
| **Endpoint Controller** | Updates Endpoint objects (service → pod mapping) |
| **Namespace Controller** | Cleans up deleted namespaces |
| **ServiceAccount Controller** | Creates default service accounts |
| **Job Controller** | Manages batch jobs |

**How it works:**
```
Current State vs Desired State = Action

Example: Deployment wants 3 replicas
Current: 2 pods running
Action: Create 1 more pod

Current: 4 pods running
Action: Delete 1 pod

Current: 3 pods running
Action: Do nothing (steady state)
```

**Loop Pattern:**
```python
while True:
    desired_state = get_from_etcd()
    current_state = get_current_cluster_state()
    
    if desired_state != current_state:
        take_action_to_reconcile()
    
    sleep(5_seconds)
```

---

### 1.5 cloud-controller-manager ☁️

**Role:** Links Kubernetes to cloud provider APIs

**What it does:**
- Separates cloud-specific logic from core Kubernetes
- Enables integration with AWS, Azure, GCP, etc.

**Controllers:**
- **Node Controller**: Checks cloud provider for deleted nodes
- **Route Controller**: Sets up routes in cloud infrastructure
- **Service Controller**: Creates/manages cloud load balancers
- **Volume Controller**: Manages cloud storage volumes

**When is it used?**
- ✅ Running on AWS, Azure, GCP, etc.
- ❌ Running on-premises (no cloud provider)

---

## Part 2: Worker Nodes

Worker nodes are where your applications actually run.

### 2.1 kubelet 🏗️

**Role:** Agent that runs on each node

**What it does:**
- Registers the node with the API server
- Watches for pod assignments (via API server)
- Ensures containers are running in pods
- Reports node/pod status back to API server
- Executes lifecycle hooks

**Key Responsibilities:**
```
1. Receive PodSpec (pod definition) from API Server
2. Create containers (via container runtime)
3. Mount volumes
4. Run health checks (liveness/readiness probes)
5. Report status back to API Server
6. Restart failed containers (within pod restart policy)
```

**Communication Flow:**
```
API Server
    ↓ (creates Pod object)
kubelet (on Node)
    ↓ (reads PodSpec)
Container Runtime (Docker/containerd)
    ↓ (starts container)
Container Running
    ↓ (status update)
kubelet → API Server
```

**Important:**
- Kubelet does NOT manage containers not created by Kubernetes
- Communicates with container runtime via CRI (Container Runtime Interface)

---

### 2.2 kube-proxy 🌐

**Role:** Network proxy on each node

**What it does:**
- Maintains network rules on nodes
- Enables communication between services and pods
- Handles load balancing for services
- Implements Service abstraction

**How it works:**
```
Traditional approach:
  Pod A → knows IP of Pod B → direct connection
  Problem: Pod B's IP changes when it restarts!

kube-proxy approach:
  Pod A → Service VIP (virtual IP) → kube-proxy rules → Pod B
  Benefit: Stable IP even if backend pods change!
```

**Modes of Operation:**

1. **iptables mode** (default):
   - Uses Linux iptables rules
   - Good performance for small-medium clusters
   - O(n) complexity for service updates

2. **IPVS mode**:
   - Uses Linux IP Virtual Server
   - Better for large clusters with many services
   - O(1) complexity, supports more sophisticated LB algorithms

3. **Userspace mode** (legacy):
   - Older, slower method
   - Rarely used now

**Example Service Routing:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

kube-proxy creates rules so that traffic to `my-service:80` is forwarded to any pod with label `app: my-app` on port 8080.

---

### 2.3 Container Runtime 🏃

**Role:** Software responsible for running containers

**Options:**
1. **containerd** (most common, CNCF graduated)
2. **CRI-O** (lightweight, Kubernetes-native)
3. **Docker Engine** (deprecated for K8s, but still works)

**Container Runtime Interface (CRI):**
```
Kubernetes (kubelet)
      ↓ (CRI - standard interface)
Container Runtime (containerd/CRI-O/Docker)
      ↓ (OCI - Open Container Initiative)
Container
```

**Why the abstraction?**
- Kubernetes doesn't care which runtime you use
- Easy to swap runtimes without changing Kubernetes
- Promotes competition and innovation

---

### 2.4 Additional Node Components

#### cni-net (Container Network Interface)
- Provides networking for containers
- Examples: Calico, Flannel, Weave, Cilium
- Assigns IPs to pods
- Enables pod-to-pod communication across nodes

#### kube-storage-version-migrator
- Migrates storage versions for CRDs

---

## Part 3: Add-ons (Cluster DNS, UI, Monitoring)

These are not core components but essential for production:

### CoreDNS
- Provides DNS service for the cluster
- Allows pods to discover each other by name
- Runs as a deployment in kube-system namespace

```bash
# Inside cluster DNS resolution
ping my-service.default.svc.cluster.local
```

### Dashboard
- Web-based UI for managing clusters
- Alternative to kubectl for visual learners

### Metrics Server
- Collects resource metrics from kubelets
- Enables `kubectl top` commands
- Required for Horizontal Pod Autoscaler (HPA)

---

## Part 4: Component Communication

### Complete Flow: Creating a Pod

Let's trace what happens when you run:
```bash
kubectl run nginx --image=nginx --replicas=3
```

**Step-by-step:**

```
1. User executes kubectl command
         ↓
2. kubectl sends REST request to API Server
         ↓
3. API Server authenticates & authorizes request
         ↓
4. API Server validates the request
         ↓
5. API Server writes Deployment object to etcd
         ↓
6. Deployment Controller (in Controller Manager) sees new Deployment
         ↓
7. Controller creates ReplicaSet with 3 replicas
         ↓
8. Scheduler sees 3 pending pods (no node assigned)
         ↓
9. Scheduler evaluates nodes and assigns each pod to a node
         ↓
10. Scheduler updates pod specs with node assignment → API Server → etcd
         ↓
11. kubelet on each node sees assigned pods
         ↓
12. kubelet instructs container runtime to pull image and start container
         ↓
13. Container starts, kubelet reports status to API Server
         ↓
14. API Server updates etcd with pod status
         ↓
15. kubectl get pods shows "Running" status
```

**Visual Flow:**
```
┌──────────┐
│  kubectl │
└────┬─────┘
     │ POST /deployments
     ↓
┌─────────────────┐
│   API Server    │
└────────┬────────┘
         │ writes
         ↓
┌─────────────────┐
│      etcd       │
└────────┬────────┘
         │ watches
         ↓
┌─────────────────┐      ┌──────────────────┐
│  Controller     │────→ │    Scheduler     │
│     Manager     │      └────────┬─────────┘
└─────────────────┘               │ assigns node
                                  ↓
                          ┌──────────────────┐
                          │   API Server     │
                          └────────┬─────────┘
                                   │ writes
                                   ↓
                          ┌──────────────────┐
                          │      etcd        │
                          └────────┬─────────┘
                                   │ watches
                                   ↓
┌──────────────────────────────────────────────────────────┐
│                    Worker Nodes                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   kubelet   │  │   kubelet   │  │   kubelet   │      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │
│         │                │                │             │
│         ↓                ↓                ↓             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │  container  │  │  container  │  │  container  │      │
│  │   runtime   │  │   runtime   │  │   runtime   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└──────────────────────────────────────────────────────────┘
```

---

## Part 5: High Availability Architecture

### Production Multi-Master Setup

```
┌─────────────────────────────────────────────────────────┐
│                   Load Balancer                         │
│              (nginx/HAProxy/Cloud LB)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ↓               ↓               ↓
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Master Node 1│ │ Master Node 2│ │ Master Node 3│
│ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
│ │API Server│ │ │ │API Server│ │ │ │API Server│ │
│ └────┬─────┘ │ │ └────┬─────┘ │ │ └────┬─────┘ │
│ ┌────┴─────┐ │ │ ┌────┴─────┐ │ │ ┌────┴─────┐ │
│ │Scheduler │ │ │ │Scheduler │ │ │ │Scheduler │ │
│ │Controller│ │ │ │Controller│ │ │ │Controller│ │
│ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
            ┌───────────┴───────────┐
            ↓                       ↓
      ┌───────────┐           ┌───────────┐
      │  etcd 1   │◄─────────►│  etcd 2   │
      └───────────┘           └───────────┘
            ▲                       ▲
            │                       │
            └───────────┬───────────┘
                        │
                  ┌───────────┐
                  │  etcd 3   │
                  └───────────┘
                        
            ┌───────────┴───────────┐
            ↓                       ↓
      ┌───────────┐           ┌───────────┐
      │Worker 1   │           │Worker 2   │
      └───────────┘           └───────────┘
```

**Key Points:**
- Multiple API Servers behind load balancer
- Odd number of etcd nodes (3, 5, 7) for quorum
- Can lose (n-1)/2 etcd nodes and still function
- Example: 3-node etcd can lose 1 node

---

## Part 6: Namespace Isolation

Namespaces provide virtual clusters within a physical cluster:

```
Physical Cluster
├── Namespace: default
│   ├── Pods
│   ├── Services
│   └── Deployments
├── Namespace: kube-system
│   ├── CoreDNS
│   ├── Metrics Server
│   └── System components
├── Namespace: development
│   └── Dev team resources
├── Namespace: production
│   └── Prod team resources
└── Namespace: monitoring
    └── Prometheus, Grafana
```

**Benefits:**
- Resource isolation (quotas, limits)
- Access control (RBAC per namespace)
- Logical separation of environments

---

## Summary Table

| Component | Location | Purpose | Critical? |
|-----------|----------|---------|-----------|
| **API Server** | Control Plane | Front-end, authentication, validation | 🔴 Critical |
| **etcd** | Control Plane | Cluster state storage | 🔴 Critical |
| **Scheduler** | Control Plane | Pod-to-node assignment | 🟡 Important |
| **Controller Manager** | Control Plane | Maintain desired state | 🟡 Important |
| **kubelet** | Worker Node | Manage containers on node | 🔴 Critical |
| **kube-proxy** | Worker Node | Network routing & services | 🟡 Important |
| **Container Runtime** | Worker Node | Run containers | 🔴 Critical |
| **CNI Plugin** | Worker Node | Pod networking | 🟡 Important |

---

## 🧠 Knowledge Check

**Q1:** Which component is the ONLY one that can write to etcd?

**Q2:** What happens if the Scheduler goes down?

**Q3:** Why do we need an odd number of etcd nodes?

**Q4:** What's the difference between kubelet and kube-proxy?

**Q5:** Trace the journey of a pod from `kubectl run` to "Running" status.

**Q6:** Why can't pods communicate directly without kube-proxy?

**Q7:** What would happen if etcd is lost without a backup?

---

**Answers:**
<details>
<summary>Click to reveal answers</summary>

**A1:** The API Server. All other components read/write through the API Server.

**A2:** New pods without node assignments will remain pending. Existing pods continue running normally. The scheduler is stateless, so restarting it resumes scheduling.

**A3:** For quorum-based consensus. With n nodes, you can tolerate (n-1)/2 failures. 3 nodes can lose 1, 5 nodes can lose 2, etc. Even numbers don't improve fault tolerance.

**A4:** kubelet manages the lifecycle of containers on its node (start, stop, monitor). kube-proxy manages network rules for service discovery and load balancing across the node.

**A5:** kubectl → API Server (auth/validate) → etcd (store) → Controller Manager (creates ReplicaSet) → Scheduler (assigns node) → etcd (update) → kubelet (sees assignment) → Container Runtime (pulls image, starts container) → kubelet reports status → API Server → etcd → kubectl shows "Running".

**A6:** Pod IPs change when they restart. kube-proxy provides stable Virtual IPs (VIPs) for Services and maintains routing rules so clients don't need to know individual pod IPs.

**A7:** Complete cluster failure. etcd contains all cluster state. Without it, the API Server has no data, and the cluster cannot function. This is why regular backups are critical!

</details>

## 🚀 What's Next?

Now that you understand the architecture, let's set up your lab environment:

👉 **Next:** [Section 03: Setting Up Your Lab Environment](./03-setup-lab.md)
