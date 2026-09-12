# Section 05: Deployments - The Heart of Kubernetes Applications

## 🎯 Learning Objectives
By the end of this section, you will be able to:
- Understand what Deployments are and why they're essential
- Create and manage Deployments with YAML manifests
- Perform rolling updates and rollbacks safely
- Configure update strategies (RollingUpdate vs Recreate)
- Scale applications horizontally with ease
- Debug common Deployment issues
- Implement best practices for production deployments

## 📋 Table of Contents
1. [Why Deployments?](#why-deployments)
2. [Deployment Architecture](#deployment-architecture)
3. [Creating Your First Deployment](#creating-your-first-deployment)
4. [Scaling Deployments](#scaling-deployments)
5. [Updating Applications](#updating-applications)
6. [Rollback Strategies](#rollback-strategies)
7. [Update Strategies Deep Dive](#update-strategies-deep-dive)
8. [Common Patterns](#common-patterns)
9. [Troubleshooting Deployments](#troubleshooting-deployments)
10. [Best Practices](#best-practices)
11. [Hands-on Lab](#hands-on-lab)
12. [Knowledge Check](#knowledge-check)

---

## Why Deployments?

### The Problem with Pods Alone

In the previous section, we learned about **Pods**. While Pods are the basic unit of deployment, they have significant limitations:

```bash
# What happens if a Pod dies?
kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
my-app-pod    1/1     Running   0          5m

# Simulate failure
kubectl delete pod my-app-pod

# The Pod is GONE forever! 😱
kubectl get pods
No resources found in default namespace.
```

**Problems with managing Pods directly:**
- ❌ No automatic restart on failure
- ❌ No scaling capabilities
- ❌ No zero-downtime updates
- ❌ No rollback mechanism
- ❌ No declarative desired state
- ❌ Manual intervention required for everything

### The Deployment Solution

**Deployments** solve all these problems by providing:

✅ **Self-healing**: Automatically replaces failed Pods  
✅ **Scaling**: Easy horizontal scaling up/down  
✅ **Rolling Updates**: Update applications without downtime  
✅ **Rollbacks**: Revert to previous versions instantly  
✅ **Declarative Management**: Define desired state, let Kubernetes handle it  
✅ **ReplicaSets**: Manages Pod replicas automatically  

> 💡 **Key Insight**: You should almost NEVER create Pods directly in production. Always use Deployments (or other controllers like StatefulSets, DaemonSets).

---

## Deployment Architecture

### How Deployments Work

```
┌─────────────────────────────────────────────────────────┐
│                    DEPLOYMENT                           │
│  Name: my-app-deployment                                │
│  Desired Replicas: 3                                    │
│  Template: nginx:1.21                                   │
│  Strategy: RollingUpdate                                │
└─────────────────┬───────────────────────────────────────┘
                  │ Creates & Manages
                  ▼
┌─────────────────────────────────────────────────────────┐
│                    REPLICASET                           │
│  Name: my-app-deployment-7d9c6f8b5                      │
│  Desired Replicas: 3                                    │
│  Current Replicas: 3                                    │
│  Selector: app=my-app                                   │
└─────────────────┬───────────────────────────────────────┘
                  │ Ensures
                  ▼
┌─────────────────────────────────────────────────────────┐
│         PODS (Managed by ReplicaSet)                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Pod 1    │  │ Pod 2    │  │ Pod 3    │              │
│  │ nginx    │  │ nginx    │  │ nginx    │              │
│  │ Running  │  │ Running  │  │ Running  │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

### The Hierarchy Explained

1. **Deployment** (Top Level)
   - Declarative definition of desired state
   - Manages ReplicaSets
   - Handles updates and rollbacks
   - You interact with this layer

2. **ReplicaSet** (Middle Layer)
   - Automatically created by Deployment
   - Ensures correct number of Pod replicas running
   - Names include hash (e.g., `my-app-7d9c6f8b5`)
   - Rarely modified directly

3. **Pods** (Bottom Layer)
   - Actual running containers
   - Created/destroyed by ReplicaSet
   - Ephemeral and replaceable

### Real-World Analogy

Think of a Deployment like a **restaurant manager**:
- **Manager (Deployment)**: Says "I need 3 chefs working"
- **Supervisor (ReplicaSet)**: Ensures exactly 3 chefs are present
- **Chefs (Pods)**: Actually do the cooking

If a chef gets sick (Pod crashes):
- Supervisor notices and hires a replacement
- Manager doesn't need to intervene
- Restaurant keeps running smoothly

---

## Creating Your First Deployment

### Basic Deployment YAML

Let's create a simple NGINX deployment:

```yaml
# deployment-basic.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
    environment: development
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: "1.21"
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
```

### Key Fields Explained

| Field | Purpose | Required? |
|-------|---------|-----------|
| `apiVersion` | API version (apps/v1 for Deployments) | ✅ Yes |
| `kind` | Resource type (Deployment) | ✅ Yes |
| `metadata.name` | Unique name for the Deployment | ✅ Yes |
| `metadata.labels` | Labels for organization/filtering | Recommended |
| `spec.replicas` | Number of Pod replicas | ✅ Yes |
| `spec.selector.matchLabels` | Labels to identify managed Pods | ✅ Yes |
| `spec.template` | Pod template specification | ✅ Yes |
| `spec.template.metadata.labels` | Labels applied to Pods | ✅ Yes |
| `spec.template.spec.containers` | Container definitions | ✅ Yes |

### ⚠️ Critical Rule: Label Matching

The `spec.selector.matchLabels` **MUST** match `spec.template.metadata.labels`:

```yaml
spec:
  selector:
    matchLabels:
      app: nginx  # ← Must match below
  template:
    metadata:
      labels:
        app: nginx  # ← This MUST match selector
```

If they don't match, Kubernetes will reject the Deployment with:
```
error: spec.selector does not match spec.template.metadata.labels
```

### Deploy It!

```bash
# Apply the deployment
kubectl apply -f deployment-basic.yaml

# Check deployment status
kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s

# Check ReplicaSet created
kubectl get replicasets
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-7d9c6f8b5    3         3         3       30s

# Check Pods
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7d9c6f8b5-abc12    1/1     Running   0          30s
nginx-deployment-7d9c6f8b5-def34    1/1     Running   0          30s
nginx-deployment-7d9c6f8b5-ghi56    1/1     Running   0          30s

# View detailed information
kubectl describe deployment nginx-deployment
```

### Understanding the Output

```bash
kubectl describe deployment nginx-deployment
```

Key sections to watch:
```
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Mon, 12 Sep 2024 10:00:00 +0000
Labels:                 app=nginx
                        environment=development
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
           version=1.21
  Containers:
   nginx:
    Image:      nginx:1.21
    Port:       80/TCP
    Limits:
      cpu:     200m
      memory:  128Mi
    Requests:
      cpu:     100m
      memory:  64Mi
    Liveness:  http-get http://:80/ delay=15s timeout=1s period=20s
    Readiness: http-get http://:80/ delay=5s timeout=1s period=10s
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-7d9c6f8b5 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-7d9c6f8b5 to 3
```

---

## Scaling Deployments

### Why Scale?

- 📈 Handle increased traffic
- 📉 Reduce costs during low usage
- 🔄 Maintain availability during updates

### Three Ways to Scale

#### Method 1: Imperative Command (Quick)

```bash
# Scale to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Verify
kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   5/5     5            5           10m

# Scale down to 2
kubectl scale deployment nginx-deployment --replicas=2
```

#### Method 2: Edit Manifest (Declarative)

```bash
# Edit the deployment
kubectl edit deployment nginx-deployment

# Change spec.replicas from 3 to 5
# Save and exit - Kubernetes applies changes automatically
```

#### Method 3: Update YAML File (Best Practice)

```yaml
# deployment-scaling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5  # Changed from 3 to 5
  # ... rest of config
```

```bash
# Apply the change
kubectl apply -f deployment-scaling.yaml

# Watch the scaling happen in real-time
kubectl get pods -w
```

### Autoscaling (HPA - Horizontal Pod Autoscaler)

For automatic scaling based on CPU/Memory:

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# Apply HPA
kubectl apply -f hpa.yaml

# Check HPA status
kubectl get hpa
NAME        REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS
nginx-hpa   Deployment/nginx-deployment   15%/80%   2         10        2
```

> ⚠️ **Note**: HPA requires Metrics Server to be installed in your cluster. Most cloud providers include it by default.

---

## Updating Applications

### The Rolling Update Process

Kubernetes performs rolling updates by:
1. Creating new Pods with updated version
2. Waiting for them to become ready
3. Terminating old Pods
4. Repeating until all Pods are updated

This ensures **zero downtime**!

### Performing an Update

#### Method 1: Update Image Version

```bash
# Update to nginx:1.22
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Watch the rollout
kubectl rollout status deployment/nginx-deployment

# Check rollout history
kubectl rollout history deployment/nginx-deployment
```

#### Method 2: Update YAML File

```yaml
# deployment-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: "1.22"  # Updated version label
    spec:
      containers:
      - name: nginx
        image: nginx:1.22  # Updated image
        # ... rest of config
```

```bash
# Apply the update
kubectl apply -f deployment-v2.yaml

# Monitor progress
kubectl get pods -w
```

### Watching the Rollout

```bash
# In one terminal, watch pods
kubectl get pods -w

# In another terminal, trigger update
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
```

You'll see output like:
```
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-7d9c6f8b5-abc12    1/1     Running             0          15m
nginx-deployment-7d9c6f8b5-def34    1/1     Running             0          15m
nginx-deployment-7d9c6f8b5-ghi56    1/1     Running             0          15m
nginx-deployment-9f8e7d6c5-xyz89    0/1     ContainerCreating   0          1s
nginx-deployment-9f8e7d6c5-xyz89    1/1     Running             0          5s
nginx-deployment-7d9c6f8b5-abc12    1/1     Terminating         0          15m
nginx-deployment-7d9c6f8b5-abc12    0/1     Terminating         0          15m
nginx-deployment-7d9c6f8b5-abc12    0/1     Terminating         0          15m
```

Notice:
- New Pod created first (`9f8e7d6c5-xyz89`)
- Waits until ready
- Then old Pod terminated (`7d9c6f8b5-abc12`)
- Service remains available throughout!

### Rollout Status and History

```bash
# Check if rollout completed successfully
kubectl rollout status deployment/nginx-deployment
deployment "nginx-deployment" successfully rolled out

# View rollout history
kubectl rollout history deployment/nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=deployment-basic.yaml
2         kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# View details of specific revision
kubectl rollout history deployment/nginx-deployment --revision=2
```

---

## Rollback Strategies

### When to Rollback?

- 🐛 Bug discovered in new version
- ⚠️ Performance degradation
- 🔒 Security vulnerability found
- ❌ Health checks failing
- 📉 Error rates increasing

### Performing a Rollback

```bash
# Rollback to previous version (revision 1)
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Monitor rollback
kubectl rollout status deployment/nginx-deployment

# Verify current revision
kubectl rollout history deployment/nginx-deployment
```

### Rollback Example Scenario

```bash
# Current state: v1.22 running (revision 2)
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-9f8e7d6c5-aaa      1/1     Running   0          5m
nginx-deployment-9f8e7d6c5-bbb      1/1     Running   0          5m
nginx-deployment-9f8e7d6c5-ccc      1/1     Running   0          5m

# Oh no! Users reporting errors with v1.22!
# Let's check logs
kubectl logs nginx-deployment-9f8e7d6c5-aaa
[error] Configuration issue detected...

# Quick rollback!
kubectl rollout undo deployment/nginx-deployment
deployment.apps/nginx-deployment rolled back

# Watch it happen
kubectl get pods -w
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-9f8e7d6c5-aaa      1/1     Running             0          6m
nginx-deployment-9f8e7d6c5-bbb      1/1     Running             0          6m
nginx-deployment-9f8e7d6c5-ccc      1/1     Running             0          6m
nginx-deployment-7d9c6f8b5-new1     0/1     ContainerCreating   0          1s
nginx-deployment-7d9c6f8b5-new1     1/1     Running             0          5s
nginx-deployment-9f8e7d6c5-aaa      1/1     Terminating         0          6m

# Back to v1.21 (revision 1) - users happy again! ✅
```

### Pause and Resume Rollouts

If you want to update but verify before completing:

```bash
# Start rollout but pause it
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
kubectl rollout pause deployment/nginx-deployment

# Check status - will show paused
kubectl rollout status deployment/nginx-deployment

# Test the new pods manually
kubectl get pods
# Run tests, check logs, etc.

# If good, resume to complete rollout
kubectl rollout resume deployment/nginx-deployment

# If bad, rollback instead
kubectl rollout undo deployment/nginx-deployment
```

---

## Update Strategies Deep Dive

Kubernetes supports two update strategies:

### 1. RollingUpdate (Default)

Updates Pods gradually without downtime.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Can create 25% extra pods during update
      maxUnavailable: 25%  # Can have 25% pods unavailable during update
```

**How it works:**
- With 4 replicas, 25% = 1 pod
- During update:
  - Can have up to 5 pods (4 + 1 surge)
  - At least 3 pods must remain available (4 - 1 unavailable)
- Gradually replaces old with new

**Timeline for 4 replicas:**
```
Time 0:  [O][O][O][O]  - All old pods running
Time 1:  [O][O][O][N]  - Create 1 new, still have 4 available
Time 2:  [O][O][N][N]  - Terminate 1 old, create 1 new
Time 3:  [O][N][N][N]  - Continue pattern
Time 4:  [N][N][N][N]  - All new pods running
```

**When to use:**
- ✅ Production applications requiring zero downtime
- ✅ Web servers, APIs, microservices
- ✅ Most stateless applications

### 2. Recreate

Terminates all old Pods before creating new ones.

```yaml
spec:
  strategy:
    type: Recreate
```

**How it works:**
- Kills ALL existing Pods
- Then creates ALL new Pods
- Results in temporary downtime

**Timeline for 4 replicas:**
```
Time 0:  [O][O][O][O]  - All old pods running
Time 1:  [ ][ ][ ][ ]  - All terminated (DOWNTIME!)
Time 2:  [N][N][N][N]  - All new pods created
```

**When to use:**
- ⚠️ Applications that can't run multiple versions simultaneously
- ⚠️ Database migrations requiring exclusive access
- ⚠️ Development/testing environments
- ❌ NOT recommended for production web services

### Customizing RollingUpdate Parameters

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Only 1 extra pod at a time
      maxUnavailable: 0    # ZERO downtime allowed!
  template:
    # ... pod template
```

**Use cases for different settings:**

| Scenario | maxSurge | maxUnavailable | Rationale |
|----------|----------|----------------|-----------|
| Critical production | 1 | 0 | Zero downtime, slow but safe |
| Standard web app | 25% | 25% | Balanced speed and availability |
| Fast updates OK | 50% | 50% | Quick updates, some risk |
| Resource constrained | 1 | 1 | Minimize resource spike |
| High availability | 3 | 0 | Extra capacity during update |

---

## Common Patterns

### Pattern 1: Blue-Green Deployment (Manual)

While Kubernetes has rolling updates, sometimes you want blue-green:

```yaml
# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:2.0

---
# Service points to blue initially
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Switch to 'green' when ready
  ports:
  - port: 80
    targetPort: 8080
```

**Process:**
1. Deploy green alongside blue
2. Test green thoroughly
3. Update Service selector to point to green
4. Monitor
5. Delete blue if successful

### Pattern 2: Canary Deployment

Gradually shift traffic to new version:

```yaml
# Stable version (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# Canary version (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: myapp
        image: myapp:2.0

---
# Single service selects both
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp  # Selects BOTH stable and canary
  ports:
  - port: 80
    targetPort: 8080
```

Traffic distribution: 90% to stable (9 pods), 10% to canary (1 pod)

**Gradual rollout:**
```bash
# If canary looks good, increase replicas
kubectl scale deployment myapp-canary --replicas=3
kubectl scale deployment myapp-stable --replicas=7

# Eventually, make canary the new stable
kubectl set image deployment/myapp-stable myapp=myapp:2.0
kubectl delete deployment myapp-canary
```

### Pattern 3: Zero-Downtime Deployment Checklist

Ensure truly zero downtime:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-downtime-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 30  # Wait 30s after pod ready before continuing
  template:
    spec:
      containers:
      - name: app
        image: myapp:2.0
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "30"]  # Graceful shutdown
      terminationGracePeriodSeconds: 60  # Allow time for preStop
```

**Key elements:**
- ✅ `maxUnavailable: 0` - Never reduce available pods
- ✅ `readinessProbe` - Don't send traffic until ready
- ✅ `minReadySeconds` - Ensure stability before continuing
- ✅ `preStop hook` - Graceful connection draining
- ✅ `terminationGracePeriodSeconds` - Time for cleanup

---

## Troubleshooting Deployments

### Issue 1: Deployment Stuck in Progress

```bash
# Check status
kubectl rollout status deployment/myapp
Waiting for deployment "myapp" rollout to finish: 1 out of 3 new replicas have been updated...

# Describe deployment for events
kubectl describe deployment myapp

# Common causes:
# - New pods failing readiness probes
# - Resource quotas exceeded
# - Image pull errors
# - Scheduling issues

# Check new pods
kubectl get pods -l app=myapp
kubectl describe pod <new-pod-name>

# If stuck, rollback
kubectl rollout undo deployment/myapp
```

### Issue 2: Pods CrashLoopBackOff After Update

```bash
kubectl get pods
NAME                       READY   STATUS             RESTARTS   AGE
myapp-abc123               0/1     CrashLoopBackOff   5          10m

# Check logs
kubectl logs myapp-abc123
Error: Configuration file not found

# Check previous version logs (if container restarted)
kubectl logs myapp-abc123 --previous

# Describe pod for events
kubectl describe pod myapp-abc123

# Solutions:
# - Fix configuration in ConfigMap/Secret
# - Correct environment variables
# - Verify image tag exists
# - Rollback to working version
kubectl rollout undo deployment/myapp
```

### Issue 3: Insufficient Resources

```bash
kubectl describe deployment myapp
Events:
  Type     Reason            Age   From                   Message
  ----     ------            ----  ----                   -------
  Warning  FailedCreate      2m    replicaset-controller  Error creating: pods "myapp-xyz" is forbidden: exceeded quota: compute-resources

# Check resource quotas
kubectl describe quota

# Check node resources
kubectl top nodes
kubectl describe node <node-name>

# Solutions:
# - Reduce resource requests/limits
# - Scale down other deployments
# - Add more nodes to cluster
# - Request quota increase
```

### Issue 4: ImagePullBackOff

```bash
kubectl get pods
NAME                READY   STATUS             RESTARTS   AGE
myapp-abc123        0/1     ImagePullBackOff   0          5m

kubectl describe pod myapp-abc123
Events:
  Warning  Failed     1m   kubelet  Error: ImagePullBackOff

# Check image name and tag
kubectl get deployment myapp -o yaml | grep image

# Common fixes:
# - Verify image exists: docker pull <image>
# - Check image tag spelling
# - Verify registry credentials (imagePullSecrets)
# - Ensure network connectivity to registry
```

### Debugging Commands Cheat Sheet

```bash
# View deployment configuration
kubectl get deployment myapp -o yaml

# View rollout history
kubectl rollout history deployment/myapp

# Check specific revision
kubectl rollout history deployment/myapp --revision=2

# Watch rollout in real-time
kubectl rollout status deployment/myapp -w

# Pause rollout for inspection
kubectl rollout pause deployment/myapp

# Resume rollout
kubectl rollout resume deployment/myapp

# Undo last rollout
kubectl rollout undo deployment/myapp

# Undo to specific revision
kubectl rollout undo deployment/myapp --to-revision=1

# Scale deployment
kubectl scale deployment myapp --replicas=5

# Edit deployment live
kubectl edit deployment/myapp

# Restart all pods (rolling restart)
kubectl rollout restart deployment/myapp

# View pods with labels
kubectl get pods -l app=myapp

# View events related to deployment
kubectl get events --field-selector involvedObject.name=myapp
```

---

## Best Practices

### ✅ DO: Use Declarative Configuration

```yaml
# GOOD: Store in Git, apply with kubectl apply
kubectl apply -f deployment.yaml
```

```bash
# BAD: Imperative commands not tracked
kubectl create deployment myapp --image=myapp:1.0
kubectl scale deployment myapp --replicas=5
```

### ✅ DO: Set Resource Requests and Limits

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

**Why?**
- Prevents resource starvation
- Enables proper scheduling
- Allows HPA to work correctly
- Protects against runaway processes

### ✅ DO: Implement Health Checks

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3
```

**Difference:**
- **Readiness**: Is pod ready to receive traffic? (Remove from service if fails)
- **Liveness**: Is pod alive? (Restart if fails)

### ✅ DO: Use Meaningful Labels

```yaml
metadata:
  labels:
    app: payment-service
    version: v1.2.3
    environment: production
    team: payments
    cost-center: engineering
```

**Benefits:**
- Easy filtering and selection
- Better monitoring and alerting
- Cost allocation
- Access control policies

### ✅ DO: Configure Proper Update Strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
minReadySeconds: 30
```

### ✅ DO: Use Namespace Separation

```bash
# Separate environments
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Deploy to specific namespace
kubectl apply -f deployment.yaml -n production
```

### ✅ DO: Implement GitOps

```
Git Repository
     │
     │ Push to main branch
     ▼
CI/CD Pipeline
     │
     │ Validate & Test
     ▼
kubectl apply / ArgoCD
     │
     │ Deploy
     ▼
Kubernetes Cluster
```

**Tools:** ArgoCD, Flux, Jenkins X

### ❌ DON'T: Use `latest` Tag in Production

```yaml
# BAD: Unpredictable, no rollback capability
image: myapp:latest

# GOOD: Specific version
image: myapp:1.2.3
```

### ❌ DON'T: Run Single Replica in Production

```yaml
# BAD: No high availability
replicas: 1

# GOOD: Multiple replicas
replicas: 3
```

### ❌ DON'T: Skip Resource Limits

```yaml
# BAD: Can cause node instability
resources: {}

# GOOD: Defined limits
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### ❌ DON'T: Ignore Rollout Status

```bash
# BAD: Fire and forget
kubectl apply -f deployment.yaml

# GOOD: Wait and verify
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp --timeout=300s
```

---

## Hands-on Lab

### Lab 5.1: Create and Manage a Deployment

**Objective**: Create a deployment, scale it, update it, and rollback.

#### Step 1: Create Initial Deployment

```bash
# Create namespace for lab
kubectl create namespace deployment-lab

# Create deployment YAML
cat > deployment-lab.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: deployment-lab
  labels:
    app: webapp
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

# Apply deployment
kubectl apply -f deployment-lab.yaml

# Verify
kubectl get deployments -n deployment-lab
kubectl get pods -n deployment-lab
kubectl get replicasets -n deployment-lab
```

#### Step 2: Expose and Test

```bash
# Create service
kubectl expose deployment webapp --port=80 --target-port=80 -n deployment-lab

# Get service IP
kubectl get svc -n deployment-lab

# Test (from within cluster or use port-forward)
kubectl port-forward svc/webapp 8080:80 -n deployment-lab

# In another terminal
curl http://localhost:8080
```

#### Step 3: Scale Up and Down

```bash
# Scale to 5 replicas
kubectl scale deployment webapp --replicas=5 -n deployment-lab

# Verify
kubectl get pods -n deployment-lab

# Scale down to 2
kubectl scale deployment webapp --replicas=2 -n deployment-lab

# Verify
kubectl get pods -n deployment-lab
```

#### Step 4: Perform Rolling Update

```bash
# Update to nginx:1.22
kubectl set image deployment/webapp nginx=nginx:1.22 -n deployment-lab

# Watch rollout
kubectl rollout status deployment/webapp -n deployment-lab

# Check history
kubectl rollout history deployment/webapp -n deployment-lab

# Verify new version
kubectl get pods -n deployment-lab -o wide
kubectl describe pod <pod-name> -n deployment-lab | grep Image
```

#### Step 5: Simulate Failure and Rollback

```bash
# Update to non-existent version
kubectl set image deployment/webapp nginx=nginx:99.99 -n deployment-lab

# Watch it fail
kubectl rollout status deployment/webapp -n deployment-lab --timeout=60s

# Check pod status
kubectl get pods -n deployment-lab

# View error
kubectl describe pod <new-pod-name> -n deployment-lab

# Rollback
kubectl rollout undo deployment/webapp -n deployment-lab

# Verify rollback
kubectl rollout status deployment/webapp -n deployment-lab
kubectl get pods -n deployment-lab
```

#### Step 6: Advanced Update Strategy

```bash
# Edit deployment to customize update strategy
kubectl edit deployment webapp -n deployment-lab

# Change strategy section:
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# Save and exit
# Now perform another update and observe the conservative rollout
kubectl set image deployment/webapp nginx=nginx:1.23 -n deployment-lab
kubectl rollout status deployment/webapp -n deployment-lab -w
```

#### Step 7: Cleanup

```bash
kubectl delete namespace deployment-lab
```

### Lab 5.2: Zero-Downtime Deployment Challenge

**Objective**: Configure a deployment for true zero-downtime updates.

```yaml
# Create zero-downtime deployment
cat > zero-downtime.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
  namespace: default
spec:
  replicas: 5
  minReadySeconds: 30
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "30"]
      terminationGracePeriodSeconds: 60
EOF

kubectl apply -f zero-downtime.yaml

# Create continuous traffic simulation
while true; do
  curl -s http://$(kubectl get svc critical-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "localhost")
  sleep 1
done &

# Perform update while watching for failures
kubectl set image deployment/critical-app app=nginx:1.22
kubectl rollout status deployment/critical-app -w

# Did you see any failed requests? If yes, tune parameters further!
```

---

## Knowledge Check

### Quiz Questions

**Q1: What is the relationship between Deployment, ReplicaSet, and Pods?**
<details>
<summary>Click for Answer</summary>

**Answer**: Deployment manages ReplicaSets, which in turn manage Pods. The Deployment provides declarative updates and rollback capabilities, the ReplicaSet ensures the desired number of Pod replicas are running, and Pods are the actual running containers.
</details>

**Q2: What happens if `spec.selector.matchLabels` doesn't match `spec.template.metadata.labels`?**
<details>
<summary>Click for Answer</summary>

**Answer**: Kubernetes will reject the Deployment with an error. These labels MUST match because the selector needs to identify which Pods belong to this Deployment.
</details>

**Q3: What's the difference between `maxSurge` and `maxUnavailable`?**
<details>
<summary>Click for Answer</summary>

**Answer**: 
- `maxSurge`: Maximum number of EXTRA pods that can be created above the desired count during update
- `maxUnavailable`: Maximum number of pods that can be UNAVAILABLE during update

For zero downtime, set `maxUnavailable: 0`.
</details>

**Q4: How do you rollback to a specific revision?**
<details>
<summary>Click for Answer</summary>

**Answer**: 
```bash
kubectl rollout undo deployment/<name> --to-revision=<revision-number>
```
</details>

**Q5: What's the difference between readinessProbe and livenessProbe?**
<details>
<summary>Click for Answer</summary>

**Answer**:
- **readinessProbe**: Determines if pod should receive traffic. Fails = removed from service endpoints
- **livenessProbe**: Determines if pod is alive. Fails = pod is restarted

A pod can fail readiness but pass liveness (running but not ready for traffic).
</details>

**Q6: Why shouldn't you use the `latest` tag in production?**
<details>
<summary>Click for Answer</summary>

**Answer**:
- Unpredictable: Don't know which version is running
- No reproducibility: Can't recreate exact same environment
- Rollback impossible: Can't rollback to "previous latest"
- Debugging difficult: Don't know what code is running
</details>

**Q7: What does `kubectl rollout pause` do?**
<details>
<summary>Click for Answer</summary>

**Answer**: Pauses an ongoing rollout, allowing you to inspect the new pods before completing the update. Useful for canary testing or verification before full rollout.
</details>

**Q8: When would you use the Recreate strategy instead of RollingUpdate?**
<details>
<summary>Click for Answer</summary>

**Answer**: When:
- Application can't run multiple versions simultaneously
- Database schema migrations require exclusive access
- In development/test environments where downtime is acceptable
- Stateful applications needing consistent state
</details>

### Practical Exercises

1. **Exercise 1**: Create a deployment with 5 replicas, update it to a new version, pause midway, verify, then resume.

2. **Exercise 2**: Configure a deployment with custom resource limits and health checks. Verify it behaves correctly under load.

3. **Exercise 3**: Implement a canary deployment pattern with 90/10 traffic split between two versions.

4. **Exercise 4**: Break a deployment intentionally (wrong image, bad config) and practice troubleshooting and rollback.

5. **Exercise 5**: Set up HPA for a deployment and test auto-scaling under load.

---

## Summary

### Key Takeaways

✅ **Deployments** are the primary way to run applications in Kubernetes  
✅ **Never create Pods directly** in production - use Deployments  
✅ **Rolling updates** provide zero-downtime deployments  
✅ **Rollbacks** are instant and reliable  
✅ **Health checks** are critical for reliability  
✅ **Resource limits** protect cluster stability  
✅ **Labels and selectors** must match correctly  
✅ **Always track rollout status** after updates  

### Commands Cheat Sheet

```bash
# Create/Update
kubectl apply -f deployment.yaml

# View
kubectl get deployments
kubectl describe deployment <name>
kubectl get pods -l app=<label>

# Scale
kubectl scale deployment <name> --replicas=5

# Update
kubectl set image deployment/<name> <container>=<image>
kubectl apply -f new-deployment.yaml

# Monitor
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>

# Rollback
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=1

# Pause/Resume
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>

# Restart
kubectl rollout restart deployment/<name>

# Delete
kubectl delete deployment <name>
```

### What's Next?

Now that you've mastered Deployments, you need to expose them to the world! 

➡️ **Next Section**: [Services - Networking and Load Balancing](./06-services.md)

In the next section, you'll learn:
- How to expose Deployments externally
- Different Service types (ClusterIP, NodePort, LoadBalancer)
- Service discovery and DNS
- Headless Services for stateful apps
- Ingress controllers for HTTP routing
- Network Policies for security

---

## Additional Resources

- [Kubernetes Official Docs - Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Patterns - Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update-cluster/)
- [Best Practices for Production Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#best-practices)
- [Understanding Kubernetes Health Checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [GitHub Examples - Deployments](https://github.com/kubernetes/examples/tree/master/staging/deployment)

---

**🎉 Congratulations!** You've completed Section 05 on Deployments. You now have the skills to manage production-grade applications in Kubernetes with confidence!

Continue to [Section 06: Services →](./06-services.md)
