# Section 04: Pods - The Atomic Unit of Kubernetes

## 🎯 Learning Objectives
- Understand what a Pod is and why it exists
- Learn Pod architecture and multi-container patterns
- Create and manage Pods using YAML
- Understand Pod lifecycle and phases
- Master debugging techniques for Pods

## What is a Pod?

**Definition:** A Pod is the smallest deployable unit in Kubernetes that can be created and managed.

**Key Characteristics:**
- 📦 Contains one or more containers
- 🌐 Shared network namespace (same IP address)
- 💾 Shared storage volumes
- 🔄 Containers in a Pod are scheduled together
- ⏰ Ephemeral by nature (not persistent)

### Why Pods Instead of Direct Containers?

**Question:** Why doesn't Kubernetes just run containers directly like Docker?

**Answer:** Pods provide an abstraction layer that solves several problems:

#### Problem 1: Multi-Container Applications
```
Real-world scenario: You need a web server + log collector

Without Pods:
- Two separate containers
- Different IPs
- Complex networking to connect them
- Scheduled independently (might be on different nodes)

With Pods:
- Both containers in one Pod
- Same IP address
- Communicate via localhost
- Always scheduled together
```

#### Problem 2: Shared Resources
```yaml
Containers in same Pod share:
- Network namespace (same IP, port space)
- IPC namespace (inter-process communication)
- UTS namespace (hostname)
- Storage volumes
- But NOT PID namespace (process isolation)
```

#### Problem 3: Co-scheduling
```
Containers that need to be together:
- Web app + cache warmer
- App + service mesh proxy (like Istio's Envoy)
- Batch job + helper container

Pods ensure they're always on the same node
```

## Pod Architecture

```
┌─────────────────────────────────────────────┐
│                  Pod                         │
│  ┌─────────────────────────────────────┐    │
│  │         Shared Network Namespace     │    │
│  │         IP: 10.244.1.5               │    │
│  │         Hostname: my-pod             │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ┌──────────────┐    ┌──────────────┐       │
│  │  Container 1 │    │  Container 2 │       │
│  │   (App)      │    │   (Sidecar)  │       │
│  │              │    │              │       │
│  │  Port: 8080  │    │  Port: 9090  │       │
│  └──────┬───────┘    └──────┬───────┘       │
│         │                   │                │
│         └────────┬──────────┘                │
│                  │                           │
│         ┌────────▼────────┐                 │
│         │  Shared Volume  │                 │
│         │   /data/shared  │                 │
│         └─────────────────┘                 │
└─────────────────────────────────────────────┘

Communication:
- Container 1 ↔ Container 2: localhost:8080 ↔ localhost:9090
- Both containers can read/write to /data/shared
```

## Creating Your First Pod

### Method 1: Imperative (Quick Testing)
```bash
# Create a simple pod
kubectl run nginx --image=nginx:latest

# Create with specific port
kubectl run nginx --image=nginx --port=80

# Create with labels
kubectl run nginx --image=nginx --labels=app=web,tier=frontend

# Check the pod
kubectl get pods
```

### Method 2: Declarative (Recommended for Production)

Create `my-first-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: web
    tier: frontend
    environment: dev
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
    ports:
    - containerPort: 80
      protocol: TCP
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

Apply the configuration:
```bash
# Create the pod
kubectl apply -f my-first-pod.yaml

# Verify
kubectl get pods
kubectl describe pod my-app-pod
```

## Multi-Container Pods

### Use Case: Web Server + Log Collector

Create `multi-container-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logger
  labels:
    app: web
spec:
  containers:
  # Main application container
  - name: web-server
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
  
  # Sidecar container for log collection
  - name: log-collector
    image: busybox:1.36
    command: ['sh', '-c', 'while true; do cat /var/log/nginx/access.log; sleep 5; done']
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
  
  # Shared volume
  volumes:
  - name: log-volume
    emptyDir: {}
```

Apply and test:
```bash
kubectl apply -f multi-container-pod.yaml

# Check both containers are running
kubectl get pod web-with-logger

# View logs from specific container
kubectl logs web-with-logger -c web-server
kubectl logs web-with-logger -c log-collector

# Execute command in specific container
kubectl exec -it web-with-logger -c web-server -- bash
```

### Common Multi-Container Patterns

#### Pattern 1: Sidecar
```
Main container + Helper container
Example: App + Service Mesh Proxy (Envoy)
```

#### Pattern 2: Ambassador
```
Main container + Proxy container
Example: App + Database proxy
```

#### Pattern 3: Adapter
```
Main container + Normalization container
Example: App + Log format converter
```

## Pod Lifecycle

### Pod Phases

```
Pending → Running → Succeeded/Failed
    ↑       ↓
    └──←───┘
     Failed
```

| Phase | Description |
|-------|-------------|
| **Pending** | Pod accepted, containers not yet created |
| **Running** | At least one container is running/starting |
| **Succeeded** | All containers completed successfully (batch jobs) |
| **Failed** | All containers terminated with failure |
| **Unknown** | State cannot be determined |

### Container States

Each container in a pod can be in:
- **Waiting**: Not yet running (pulling image, etc.)
- **Running**: Currently executing
- **Terminated**: Finished execution (success or failure)

### View Pod Lifecycle
```bash
# Detailed pod information
kubectl describe pod my-pod

# Look for:
# - State: Running/Waiting/Terminated
# - Reason: ContainerCreating/ImagePullBackOff/CrashLoopBackOff
# - Last State: Previous termination info
# - Restart Count: How many times restarted
```

## Common Pod Issues & Debugging

### Issue 1: ImagePullBackOff
```bash
kubectl get pods
# Output: my-pod   0/1   ImagePullBackOff

# Debug:
kubectl describe pod my-pod
# Look for: "Failed to pull image"

# Solutions:
# 1. Check image name and tag
# 2. Verify image exists in registry
# 3. Check image pull secrets for private registries
# 4. Test locally: docker pull <image>
```

### Issue 2: CrashLoopBackOff
```bash
kubectl get pods
# Output: my-pod   0/1   CrashLoopBackOff   5/5

# Debug:
kubectl logs my-pod                    # Current logs
kubectl logs my-pod --previous         # Logs from crashed container
kubectl describe pod my-pod            # Events and state

# Common causes:
# 1. Application error (check logs)
# 2. Missing configuration
# 3. Failed health checks
# 4. Resource constraints (OOMKilled)
```

### Issue 3: Pending State
```bash
kubectl get pods
# Output: my-pod   0/1   Pending

# Debug:
kubectl describe pod my-pod
# Look for scheduling failures

# Common causes:
# 1. Insufficient resources (CPU/Memory)
# 2. No nodes match node selector
# 3. Taints/tolerations mismatch
# 4. PersistentVolume not available

# Check node resources:
kubectl top nodes
kubectl describe nodes
```

### Issue 4: OOMKilled (Out of Memory)
```bash
kubectl describe pod my-pod
# Look for: Reason: OOMKilled

# Solution: Increase memory limits
spec:
  containers:
  - name: app
    resources:
      limits:
        memory: "512Mi"  # Increase from 256Mi
```

## Pod Configuration Options

### Environment Variables
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Simple key-value
    - name: DATABASE_URL
      value: "postgres://localhost:5432/mydb"
    
    # From ConfigMap
    - name: API_KEY
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: api-key
    
    # From Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    
    # From Downward API (pod info)
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
```

### Health Checks (Probes)
```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    
    # Liveness Probe (restart if fails)
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    # Readiness Probe (don't send traffic if fails)
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
    
    # Startup Probe (for slow-starting apps)
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

### Resource Management
```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      # Guaranteed minimum resources
      requests:
        memory: "256Mi"
        cpu: "250m"  # 250 milliCPUs
      
      # Maximum resources allowed
      limits:
        memory: "512Mi"
        cpu: "500m"
```

**QoS Classes:**
- **Guaranteed**: Both requests and limits set, and equal
- **Burstable**: Requests set, limits higher or not set
- **BestEffort**: No requests or limits set (first to be killed)

## Working with Pods: Essential Commands

```bash
# List pods
kubectl get pods
kubectl get pods -n <namespace>
kubectl get pods --all-namespaces
kubectl get pods -o wide  # More details

# Create pod
kubectl apply -f pod.yaml
kubectl create -f pod.yaml

# Delete pod
kubectl delete pod my-pod
kubectl delete -f pod.yaml

# View logs
kubectl logs my-pod
kubectl logs my-pod -c container-name  # Multi-container
kubectl logs -f my-pod                 # Follow logs
kubectl logs --previous my-pod         # Crashed container

# Execute commands
kubectl exec my-pod -- ls /app
kubectl exec -it my-pod -- bash        # Interactive shell
kubectl exec -it my-pod -c container -- bash

# Copy files
kubectl cp ./file.txt my-pod:/app/file.txt
kubectl cp my-pod:/app/logs.txt ./logs.txt

# Describe pod (detailed info)
kubectl describe pod my-pod

# Edit running pod (limited changes)
kubectl edit pod my-pod

# Port forwarding
kubectl port-forward my-pod 8080:80
# Access via: http://localhost:8080

# View pod YAML
kubectl get pod my-pod -o yaml
kubectl get pod my-pod -o json
```

## Best Practices

### ✅ DO:
1. **Use Deployments instead of raw Pods** (for production)
2. **Set resource requests and limits**
3. **Add health probes** (liveness, readiness)
4. **Use labels for organization**
5. **Keep Pods immutable** (don't edit running pods)
6. **Use multi-container pods wisely** (only when tightly coupled)

### ❌ DON'T:
1. **Don't run stateful apps in plain Pods** (use StatefulSets)
2. **Don't create Pods imperatively in production**
3. **Don't skip resource limits** (causes resource starvation)
4. **Don't put loosely coupled containers in same Pod**
5. **Don't rely on Pod restarts for data persistence**

## Real-World Example: Complete Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-web-app
  labels:
    app: web-frontend
    version: v1.2.0
    team: platform
  annotations:
    description: "Production web application pod"
spec:
  # Security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  
  containers:
  - name: web-app
    image: myregistry/web-app:v1.2.0
    imagePullPolicy: Always
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
    
    # Environment variables
    env:
    - name: NODE_ENV
      value: "production"
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db-host
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    
    # Health checks
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    
    # Resources
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
    
    # Volume mounts
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
      readOnly: true
    - name: data-volume
      mountPath: /app/data
  
  # Init container (runs before main container)
  initContainers:
  - name: init-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nslookup $DB_HOST; do echo waiting; sleep 2; done']
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db-host
  
  # Volumes
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: data-volume
    emptyDir: {}
  
  # DNS configuration
  dnsPolicy: ClusterFirst
  dnsConfig:
    searches:
      - default.svc.cluster.local
      - svc.cluster.local
      - cluster.local
```

## Summary

| Concept | Key Point |
|---------|-----------|
| **Pod** | Smallest K8s unit, contains 1+ containers |
| **Network** | All containers share same IP |
| **Storage** | Volumes shared across containers |
| **Lifecycle** | Pending → Running → Succeeded/Failed |
| **Debugging** | Use `describe`, `logs`, `exec` |
| **Production** | Use Deployments, not raw Pods |

## 🧠 Knowledge Check

**Q1:** Why use Pods instead of running containers directly?

**Q2:** How do two containers in the same Pod communicate?

**Q3:** What's the difference between liveness and readiness probes?

**Q4:** What does CrashLoopBackOff mean and how do you debug it?

**Q5:** When should you use a multi-container Pod vs. separate Pods?

---

**Answers:**
<details>
<summary>Click to reveal answers</summary>

**A1:** Pods provide shared network/storage namespaces, co-scheduling guarantees, and abstraction for multi-container applications.

**A2:** Via localhost - they share the same network namespace and IP address.

**A3:** Liveness probe determines if container should be restarted. Readiness probe determines if container should receive traffic.

**A4:** Container keeps crashing and restarting. Debug with: `kubectl logs --previous`, `kubectl describe`, check resource limits and app errors.

**A5:** Use multi-container when containers are tightly coupled, need same lifecycle, share storage, or communicate frequently via localhost. Use separate Pods for loosely coupled services.

</details>

## 🚀 What's Next?

Now that you understand Pods, learn how to manage them at scale:

👉 **Next:** [Section 05: Deployments & ReplicaSets](./05-deployments.md)
