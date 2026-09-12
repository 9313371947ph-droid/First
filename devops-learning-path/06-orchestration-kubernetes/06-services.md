# Section 06: Services - Networking and Load Balancing in Kubernetes

## 🎯 Learning Objectives
By the end of this section, you will be able to:
- Understand Kubernetes networking fundamentals
- Create and configure different Service types
- Implement service discovery and DNS
- Use headless Services for stateful applications
- Configure Ingress for HTTP/HTTPS routing
- Implement Network Policies for security
- Troubleshoot common networking issues
- Design production-ready networking architectures

## 📋 Table of Contents
1. [Why Do We Need Services?](#why-do-we-need-services)
2. [Kubernetes Networking Model](#kubernetes-networking-model)
3. [Service Types Deep Dive](#service-types-deep-dive)
4. [Service Discovery and DNS](#service-discovery-and-dns)
5. [Headless Services](#headless-services)
6. [Ingress Controllers](#ingress-controllers)
7. [Network Policies](#network-policies)
8. [Advanced Patterns](#advanced-patterns)
9. [Troubleshooting Network Issues](#troubleshooting-network-issues)
10. [Best Practices](#best-practices)
11. [Hands-on Lab](#hands-on-lab)
12. [Knowledge Check](#knowledge-check)

---

## Why Do We Need Services?

### The Problem with Pods

Recall from previous sections that **Pods are ephemeral**:

```bash
kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
web-app-abc   1/1     Running   0          5m
web-app-def   1/1     Running   0          5m
web-app-ghi   1/1     Running   0          5m

# Get their IPs
kubectl get pods -o wide
NAME          READY   STATUS    IP           NODE
web-app-abc   1/1     Running   10.244.1.5   node-1
web-app-def   1/1     Running   10.244.2.3   node-2
web-app-ghi   1/1     Running   10.244.3.7   node-3

# Now simulate failure
kubectl delete pod web-app-abc

# New pod gets a DIFFERENT IP!
kubectl get pods -o wide
NAME          READY   STATUS    IP           NODE
web-app-def   1/1     Running   10.244.2.3   node-2
web-app-ghi   1/1     Running   10.244.3.7   node-3
web-app-jkl   1/1     Running   10.244.1.9   node-1  # ← NEW IP!
```

**Problems:**
- ❌ Pod IPs change constantly
- ❌ Pods can be on different nodes
- ❌ No load balancing between pods
- ❌ No stable endpoint for clients
- ❌ Hard to discover services dynamically

### The Service Solution

**Services** provide:

✅ **Stable IP Address**: Virtual IP (ClusterIP) that never changes  
✅ **DNS Name**: Human-readable name for service discovery  
✅ **Load Balancing**: Automatic distribution across Pods  
✅ **Service Discovery**: Find services by name  
✅ **Decoupling**: Clients don't need to know Pod IPs  
✅ **External Access**: Expose apps outside the cluster  

> 💡 **Key Insight**: Think of a Service as a "permanent load balancer" that sits in front of your dynamic Pods.

---

## Kubernetes Networking Model

### The Flat Network Assumption

Kubernetes assumes a flat network where:

1. **All Pods can communicate** without NAT
2. **All Nodes can communicate** with all Pods
3. **Pod IP is the same** as seen by others

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌─────────────┐         ┌─────────────┐                │
│  │   Node 1    │         │   Node 2    │                │
│  │             │         │             │                │
│  │  Pod A      │◄───────►│  Pod C      │  Direct IP     │
│  │  10.244.1.5 │         │  10.244.2.3 │  Communication │
│  │             │         │             │                │
│  │  Pod B      │◄───────►│  Pod D      │                │
│  │  10.244.1.6 │         │  10.244.2.4 │                │
│  │             │         │             │                │
│  └─────────────┘         └─────────────┘                │
│                                                          │
│  All Pods can reach each other directly via IP          │
└─────────────────────────────────────────────────────────┘
```

### How Services Work Internally

```
┌─────────────────────────────────────────────────────────┐
│                    Client Pod                            │
│                    10.244.1.10                           │
│                         │                                │
│                         │ Request to my-service:80       │
│                         ▼                                │
│              ┌─────────────────────┐                     │
│              │   kube-proxy        │                     │
│              │   (on each node)    │                     │
│              │                     │                     │
│              │  Maintains iptables │                     │
│              │  or IPVS rules      │                     │
│              └─────────┬───────────┘                     │
│                        │                                 │
│            ┌───────────┼───────────┐                    │
│            ▼           ▼           ▼                    │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│     │ Pod 1    │ │ Pod 2    │ │ Pod 3    │             │
│     │ 10.244.1.5│ │10.244.2.3│ │10.244.3.7│             │
│     │ :80      │ │ :80      │ │ :80      │             │
│     └──────────┘ └──────────┘ └──────────┘             │
│                                                          │
│  Service VIP: 10.96.45.123:80                           │
│  DNS: my-service.default.svc.cluster.local              │
└─────────────────────────────────────────────────────────┘
```

**kube-proxy** runs on every node and:
- Watches the Kubernetes API for Service changes
- Updates network rules (iptables/IPVS) to route traffic
- Implements load balancing algorithms

---

## Service Types Deep Dive

Kubernetes supports 4 main Service types:

### 1. ClusterIP (Default)

**Purpose**: Internal cluster communication only

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: production
spec:
  type: ClusterIP  # Default, can be omitted
  selector:
    app: backend
    tier: api
  ports:
  - name: http
    port: 80           # Service port
    targetPort: 8080   # Container port
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
```

**Characteristics:**
- ✅ Only accessible within the cluster
- ✅ Stable ClusterIP assigned automatically
- ✅ Can specify custom IP from service subnet
- ✅ Most common type for internal services

**Use Cases:**
- Frontend → Backend communication
- Microservices talking to each other
- Database access from application pods
- Monitoring scrapes

**Access Pattern:**
```bash
# From inside cluster
curl http://backend-api.production.svc.cluster.local:80
curl http://backend-api:80  # If in same namespace

# From outside cluster - NOT ACCESSIBLE! ❌
curl http://<cluster-ip>:80  # Won't work
```

### 2. NodePort

**Purpose**: Expose service on each node's IP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # Optional: 30000-32767
```

**Characteristics:**
- ✅ Exposes service on port 30000-32767 on EVERY node
- ✅ Builds on top of ClusterIP
- ✅ Accessible from outside cluster via NodeIP:NodePort
- ⚠️ Port range limited
- ⚠️ Need to manage firewall rules

**Use Cases:**
- Quick external access for development
- Simple demos and testing
- When you don't have a LoadBalancer

**Access Pattern:**
```bash
# Get node IPs
kubectl get nodes -o wide

# Access via any node
curl http://<node-ip>:30080

# Works on ALL nodes (traffic routed to pods)
curl http://node1-ip:30080
curl http://node2-ip:30080
curl http://node3-ip:30080
```

### 3. LoadBalancer

**Purpose**: Cloud provider's external load balancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: production-web
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  loadBalancerIP: 203.0.113.50  # Optional: static IP
  loadBalancerSourceRanges:     # Optional: restrict access
  - 0.0.0.0/0
```

**Characteristics:**
- ✅ Automatically provisions cloud LB (AWS ELB, GCP CLB, Azure LB)
- ✅ Public IP address provided
- ✅ External traffic → LB → NodePort → ClusterIP → Pods
- ⚠️ Cloud provider specific
- ⚠️ Costs money (each LB is billed)
- ⚠️ Takes time to provision (1-5 minutes)

**Use Cases:**
- Production applications needing external access
- High availability requirements
- Integration with cloud DNS/SSL

**Access Pattern:**
```bash
# Wait for external IP
kubectl get svc production-web
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
production-web   LoadBalancer   10.96.45.123   203.0.113.50    80:30080/TCP

# Access via external IP
curl http://203.0.113.50

# Or via DNS (if configured)
curl http://myapp.example.com
```

### 4. ExternalName

**Purpose**: Map service to external DNS name

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
  ports:
  - port: 5432
```

**Characteristics:**
- ✅ No selector (doesn't select pods)
- ✅ Returns CNAME record to external DNS
- ✅ Allows using external services like internal ones
- ✅ No proxying, just DNS redirection

**Use Cases:**
- Migrating to cloud (hybrid setup)
- Using managed databases (RDS, Cloud SQL)
- Third-party API access
- Legacy system integration

**Access Pattern:**
```bash
# Inside cluster, access external DB as if it's internal
psql -h external-db -U user database

# Resolves to: db.example.com
```

---

## Service Type Comparison

| Feature | ClusterIP | NodePort | LoadBalancer | ExternalName |
|---------|-----------|----------|--------------|--------------|
| **Internal Access** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **External Access** | ❌ No | ✅ Via NodeIP:Port | ✅ Via LB IP | ✅ Via DNS |
| **Cloud Provider** | Any | Any | Required | Any |
| **Cost** | Free | Free | $ (per LB) | Free |
| **Setup Time** | Instant | Instant | 1-5 min | Instant |
| **Selector** | Required | Required | Required | None |
| **Use Case** | Internal | Dev/Test | Production | External Services |

---

## Service Discovery and DNS

### Kubernetes DNS System

Kubernetes includes built-in DNS (CoreDNS by default):

```
┌─────────────────────────────────────────────────────────┐
│              Kubernetes DNS Hierarchy                    │
│                                                          │
│  <service>.<namespace>.svc.cluster.local                │
│       │          │        │            │                 │
│       │          │        │            └─ Cluster domain │
│       │          │        └─ Service namespace           │
│       │          └─ "svc" subdomain                      │
│       └─ Service name                                    │
│                                                          │
│  Examples:                                               │
│  - backend.default.svc.cluster.local                    │
│  - api.production.svc.cluster.local                     │
│  - redis.cache.svc.cluster.local                        │
└─────────────────────────────────────────────────────────┘
```

### DNS Resolution Rules

**Within same namespace:**
```bash
# Short name works
curl http://backend-service
curl http://backend-service:8080
```

**Different namespace:**
```bash
# Need full name or include namespace
curl http://backend-service.production
curl http://backend-service.production.svc.cluster.local
```

**From Pod to Service:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Use DNS name in environment variables
    - name: BACKEND_URL
      value: "http://backend-api:8080"
    
    # Or use service discovery
    - name: BACKEND_HOST
      value: "backend-api"
```

### Environment Variables Alternative

Kubernetes also injects environment variables for services:

```bash
# When a Pod runs, it gets env vars for existing services
env | grep BACKEND
BACKEND_API_SERVICE_HOST=10.96.45.123
BACKEND_API_SERVICE_PORT=8080
BACKEND_API_PORT=tcp://10.96.45.123:8080
BACKEND_API_PORT_8080_TCP=tcp://10.96.45.123:8080
BACKEND_API_PORT_8080_TCP_PROTO=tcp
BACKEND_API_PORT_8080_TCP_PORT=8080
BACKEND_API_PORT_8080_TCP_ADDR=10.96.45.123
```

> ⚠️ **Note**: Environment variables only include services created BEFORE the pod. DNS is preferred!

### Testing DNS

```bash
# Deploy a test pod
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes

# Output:
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-system.kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

# Test your service
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup backend-api.default

# Check DNS pod status
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

## Headless Services

### What Are Headless Services?

**Normal Service**: Has a ClusterIP, load balances to pods  
**Headless Service**: NO ClusterIP, returns all pod IPs directly

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-app
spec:
  clusterIP: None  # ← This makes it "headless"
  selector:
    app: stateful-app
  ports:
  - port: 80
    targetPort: 80
```

### Why Use Headless Services?

**Use Case 1: StatefulSets**
- Each pod needs stable network identity
- Direct pod-to-pod communication
- Leader election patterns

**Use Case 2: Custom Load Balancing**
- Client-side load balancing
- Service mesh (Istio, Linkerd)
- Advanced routing logic

**Use Case 3: Monitoring**
- Scrape all pods individually
- Collect per-pod metrics
- Avoid aggregation

### DNS Behavior Difference

**Normal Service:**
```bash
nslookup backend-api
Name:   backend-api.default.svc.cluster.local
Address: 10.96.45.123  # Single VIP

# Traffic goes to VIP, kube-proxy load balances
```

**Headless Service:**
```bash
nslookup stateful-app
Name:   stateful-app.default.svc.cluster.local
Address: 10.244.1.5   # Pod 1 IP
Address: 10.244.2.3   # Pod 2 IP
Address: 10.244.3.7   # Pod 3 IP

# Client chooses which pod to contact directly
```

### StatefulSet + Headless Service Pattern

```yaml
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql

---
# StatefulSet uses the headless service
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"  # Must match service name
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
---
# Pods get predictable names:
# mysql-0.mysql.default.svc.cluster.local
# mysql-1.mysql.default.svc.cluster.local
# mysql-2.mysql.default.svc.cluster.local
```

---

## Ingress Controllers

### Why Ingress?

**Problem with LoadBalancer Services:**
- Expensive ($20-30/month per LB on most clouds)
- One LB per service = many LBs for microservices
- Limited to L4 (TCP/UDP), not HTTP-aware
- No path-based routing
- No SSL termination management

**Solution: Ingress**

```
┌─────────────────────────────────────────────────────────┐
│                    Internet                              │
│                         │                                │
│                         ▼                                │
│              ┌─────────────────────┐                     │
│              │   LoadBalancer      │  (Only ONE!)        │
│              │   Service           │                     │
│              └─────────┬───────────┘                     │
│                        │                                 │
│                        ▼                                 │
│              ┌─────────────────────┐                     │
│              │   Ingress Controller│  (nginx, traefik)   │
│              │   - Path Routing    │                     │
│              │   - SSL Termination │                     │
│              │   - Host-based      │                     │
│              └─────────┬───────────┘                     │
│                        │                                 │
│        ┌───────────────┼───────────────┐                │
│        ▼               ▼               ▼                │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│   │ /api    │    │ /web    │    │ /admin  │            │
│   │ Service │    │ Service │    │ Service │            │
│   └─────────┘    └─────────┘    └─────────┘            │
└─────────────────────────────────────────────────────────┘
```

### Ingress vs Service

| Aspect | Service | Ingress |
|--------|---------|---------|
| **Layer** | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| **Routing** | IP:Port | Path, Host, Headers |
| **SSL** | No | Yes (termination) |
| **Cost** | Per service | One LB for many |
| **Complexity** | Simple | More configuration |

### Basic Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx  # Specify controller
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### SSL/TLS with Ingress

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 443
```

### Popular Ingress Controllers

1. **NGINX Ingress Controller** (Most popular)
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
   ```

2. **Traefik**
   - Dynamic configuration
   - Built-in Let's Encrypt
   - Dashboard included

3. **HAProxy Ingress**
   - High performance
   - Advanced routing

4. **AWS ALB Ingress**
   - Native AWS integration
   - Provisions ALB automatically

5. **GKE Ingress**
   - Google Cloud Load Balancer
   - Managed certificates

### Ingress Annotations (NGINX)

```yaml
metadata:
  annotations:
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: "basic"
    nginx.ingress.kubernetes.io/auth-secret: "auth-secret"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    
    # Redirect
    nginx.ingress.kubernetes.io/permanent-redirect: "https://new-domain.com"
    
    # Rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # SSL
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

---

## Network Policies

### Why Network Policies?

**Default Behavior**: All Pods can communicate freely
```
Pod A ─────► Pod B  ✅ Allowed
Pod A ─────► Pod C  ✅ Allowed
Pod A ─────► Internet ✅ Allowed
```

**Security Problem**: 
- Compromised pod can access everything
- No isolation between namespaces
- Violation of least privilege

**Solution**: Network Policies (firewall for Pods)

### Network Policy Basics

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # Select all pods
  policyTypes:
  - Ingress
  - Egress
  # No rules = deny all traffic
```

### Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Translation**: "Only allow frontend pods to reach backend pods on port 8080"

### Complex Policy Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
      tier: data
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow from backend pods
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow from monitoring namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9187  # Exporter port
  
  egress:
  # Allow DNS resolution
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Network Policy Types

**Ingress Rules** (incoming traffic):
```yaml
ingress:
- from:
  - podSelector:      # From specific pods
      matchLabels:
        role: frontend
  - namespaceSelector: # From specific namespace
      matchLabels:
        name: trusted
  - ipBlock:          # From specific IP range
      cidr: 10.0.0.0/8
  ports:
  - protocol: TCP
    port: 80
```

**Egress Rules** (outgoing traffic):
```yaml
egress:
- to:
  - podSelector:
      matchLabels:
        role: database
  ports:
  - protocol: TCP
    port: 5432
  
  # Allow external API access
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 10.0.0.0/8  # Except private networks
```

### CNI Plugin Requirements

⚠️ **Important**: Network Policies require a CNI plugin that supports them:

✅ **Supported**:
- Calico
- Cilium
- Weave Net
- Antrea
- Cloud providers (GKE, EKS, AKS) with network policy enabled

❌ **Not Supported**:
- Flannel (default in many setups)
- Basic kubeadm without add-on

**Enable on Cloud Providers:**

GKE:
```bash
gcloud container clusters create my-cluster --enable-network-policy
```

EKS:
```bash
# Install Calico
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-operator.yaml
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-crs.yaml
```

AKS:
```bash
az aks create --network-policy azure
```

---

## Advanced Patterns

### Pattern 1: Multi-Tier Architecture

```yaml
# Frontend - public access
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80

---
# Backend - internal only
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080

---
# Database - highly restricted
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ClusterIP
  clusterIP: 10.96.100.100  # Fixed IP
  selector:
    app: database
  ports:
  - port: 5432

---
# Network policies for security
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

### Pattern 2: Blue-Green with Services

```yaml
# Blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
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
      - name: app
        image: myapp:1.0

---
# Green deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
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
      - name: app
        image: myapp:2.0

---
# Service initially points to blue
apiVersion: v1
kind: Service
metadata:
  name: app-production
spec:
  type: LoadBalancer
  selector:
    app: myapp
    version: blue  # ← Switch to 'green' when ready
  ports:
  - port: 80
    targetPort: 8080
```

**Switch Process:**
```bash
# Test green thoroughly
kubectl port-forward deploy/app-green 8081:8080

# Switch traffic
kubectl patch service app-production -p '{"spec":{"selector":{"version":"green"}}}'

# Monitor
kubectl get svc app-production

# Delete blue if successful
kubectl delete deployment app-blue
```

### Pattern 3: Service Mesh Preparation

```yaml
# Headless service for service mesh
apiVersion: v1
kind: Service
metadata:
  name: microservice
  labels:
    app: microservice
spec:
  clusterIP: None  # Headless
  selector:
    app: microservice
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090

---
# Regular service for external access
apiVersion: v1
kind: Service
metadata:
  name: microservice-external
spec:
  type: ClusterIP
  selector:
    app: microservice
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

**Benefits:**
- Service mesh (Istio/Linkerd) handles load balancing
- Sidecar proxies intercept all traffic
- Advanced features: retries, timeouts, circuit breakers
- Observability built-in

### Pattern 4: Multi-Cluster Service Discovery

```yaml
# For multi-cluster setups
apiVersion: v1
kind: Service
metadata:
  name: global-service
  annotations:
    # Cloud-specific annotations for global LB
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: global-app
  ports:
  - port: 80
```

**Tools for Multi-Cluster:**
- Istio Multi-Cluster
- Linkerd Multi-Cluster
- Submariner
- Cloudburst

---

## Troubleshooting Network Issues

### Issue 1: Service Not Accessible

```bash
# Check service exists
kubectl get svc my-service

# Check endpoints (pods matching selector)
kubectl get endpoints my-service
# Should show pod IPs, if empty = selector mismatch!

# Verify selector matches pod labels
kubectl get svc my-service -o yaml | grep -A5 selector
kubectl get pods --show-labels | grep my-label

# Common problem: labels don't match!
```

### Issue 2: DNS Not Resolving

```bash
# Test DNS from pod
kubectl run -it --rm dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Verify DNS service
kubectl get svc -n kube-system kube-dns

# Check resolv.conf in pod
kubectl exec my-pod -- cat /etc/resolv.conf
```

### Issue 3: Connection Refused

```bash
# Check if pods are running
kubectl get pods -l app=myapp

# Check container port
kubectl describe pod my-pod | grep -A10 Ports

# Verify service targetPort matches container port
kubectl get svc my-service -o yaml | grep -A5 ports

# Test from inside cluster
kubectl run -it --rm test --image=busybox --restart=Never -- wget -O- http://my-service:80

# Check network policies blocking traffic
kubectl get networkpolicy
kubectl describe networkpolicy my-policy
```

### Issue 4: External Access Not Working

```bash
# For NodePort:
# 1. Check firewall allows 30000-32767
# 2. Verify node external IPs
kubectl get nodes -o wide

# For LoadBalancer:
# Check if external IP assigned
kubectl get svc my-lb-service

# If pending, check:
kubectl describe svc my-lb-service | grep -A10 Events

# Cloud provider specific:
# AWS: Check IAM permissions for ELB creation
# GCP: Check firewall rules
# Azure: Check resource quotas
```

### Debugging Commands Cheat Sheet

```bash
# View service details
kubectl get svc <name> -o yaml
kubectl describe svc <name>

# Check endpoints
kubectl get endpoints <name>
kubectl describe endpoints <name>

# Test connectivity from pod
kubectl run test-pod --image=busybox --rm -it --restart=Never -- \
  wget -O- http://<service-name>:<port>

# Test DNS
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup <service-name>

# Check network policies
kubectl get networkpolicy
kubectl describe networkpolicy <name>

# View iptables rules (on node)
ssh <node>
sudo iptables-save | grep <service-cluster-ip>

# Trace packet flow
kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash
# Inside: tcpdump, curl, mtr, etc.

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

---

## Best Practices

### ✅ DO: Use Meaningful Service Names

```yaml
# GOOD
metadata:
  name: payment-service
  name: user-api
  name: redis-cache

# BAD
metadata:
  name: svc1
  name: service-new
  name: test
```

### ✅ DO: Use Named Ports

```yaml
# GOOD: Self-documenting
ports:
- name: http
  port: 80
  targetPort: 8080
- name: metrics
  port: 9090
  targetPort: 9090
- name: grpc
  port: 9091
  targetPort: 9091

# BAD: Magic numbers
ports:
- port: 80
  targetPort: 8080
- port: 9090
  targetPort: 9090
```

### ✅ DO: Implement Network Policies

```yaml
# Start with deny-all, then allow specific
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Then add specific allow rules
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
```

### ✅ DO: Use Ingress for HTTP Services

```yaml
# GOOD: One LB, multiple routes
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        backend:
          service:
            name: user-service
      - path: /orders
        backend:
          service:
            name: order-service

# BAD: Multiple expensive LBs
# (One LoadBalancer service per microservice)
```

### ✅ DO: Set Resource Limits on Ingress Controller

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 1
    memory: 512Mi
```

### ✅ DO: Use TLS Everywhere

```yaml
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    # ...
```

### ❌ DON'T: Expose Database Services Externally

```yaml
# BAD: Database accessible from internet
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: LoadBalancer  # ❌ Never do this!
  selector:
    app: postgres
  ports:
  - port: 5432

# GOOD: Internal only
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP  # ✅ Internal only
  selector:
    app: postgres
  ports:
  - port: 5432
```

### ❌ DON'T: Skip Endpoint Verification

```bash
# BAD: Assume service works
kubectl apply -f service.yaml

# GOOD: Verify endpoints exist
kubectl apply -f service.yaml
kubectl get endpoints my-service
# Should show pod IPs, not "<none>"
```

### ❌ DON'T: Use NodePort in Production

```yaml
# BAD: NodePort for production
spec:
  type: NodePort  # ❌ Security risk, port management issues

# GOOD: Use Ingress or LoadBalancer
spec:
  type: ClusterIP
# Then create Ingress resource
```

---

## Hands-on Lab

### Lab 6.1: Create Services and Test Connectivity

**Objective**: Create different service types and verify connectivity.

#### Step 1: Setup Namespace and Deployment

```bash
# Create namespace
kubectl create namespace services-lab

# Deploy a simple web app
cat > webapp-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: services-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
EOF

kubectl apply -f webapp-deployment.yaml

# Verify pods
kubectl get pods -n services-lab
```

#### Step 2: Create ClusterIP Service

```bash
# Create ClusterIP service
cat > clusterip-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: webapp-clusterip
  namespace: services-lab
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 80
EOF

kubectl apply -f clusterip-svc.yaml

# Get service IP
kubectl get svc webapp-clusterip -n services-lab

# Test from within cluster
kubectl run -it --rm test-pod --image=busybox --namespace=services-lab --restart=Never -- \
  wget -O- http://webapp-clusterip:80
```

#### Step 3: Create NodePort Service

```bash
# Create NodePort service
cat > nodeport-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
  namespace: services-lab
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
EOF

kubectl apply -f nodeport-svc.yaml

# Get node IPs
kubectl get nodes -o wide

# Access from outside (replace with actual node IP)
curl http://<node-ip>:30080
```

#### Step 4: Test DNS Resolution

```bash
# Test DNS from pod
kubectl run -it --rm dns-test --image=busybox:1.28 \
  --namespace=services-lab --restart=Never -- \
  nslookup webapp-clusterip

# Test full DNS name
kubectl run -it --rm dns-test --image=busybox:1.28 \
  --namespace=services-lab --restart=Never -- \
  nslookup webapp-clusterip.services-lab.svc.cluster.local
```

### Lab 6.2: Implement Network Policies

**Objective**: Restrict traffic between pods using Network Policies.

```bash
# Create namespaces
kubectl create namespace frontend
kubectl create namespace backend

# Deploy frontend app
cat > frontend.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "3600"]
EOF

kubectl apply -f frontend.yaml

# Deploy backend app
cat > backend.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
EOF

kubectl apply -f backend.yaml

# Test connectivity before policy
kubectl exec -n frontend deploy/frontend -- wget -O- http://backend-svc.backend:80

# Apply network policy
cat > network-policy.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external-ingress
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

kubectl apply -f network-policy.yaml

# Test after policy - should still work (from frontend namespace)
kubectl exec -n frontend deploy/frontend -- wget -O- http://backend-svc.backend:80

# Test from other namespace - should fail
kubectl create namespace other
kubectl run test-pod --image=busybox --namespace=other --restart=Never --rm -it -- \
  wget -O- --timeout=2 http://backend-svc.backend:80
# Should timeout/fail
```

### Lab 6.3: Configure Ingress

**Objective**: Set up path-based routing with Ingress.

```bash
# Install NGINX Ingress Controller (if not present)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Wait for installation
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Deploy multiple services
cat > multi-service.yaml <<EOF
# API Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        command: ["/bin/sh", "-c"]
        args:
        - echo "API Response" > /usr/share/nginx/html/index.html; nginx -g "daemon off;"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
---
# Web Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        command: ["/bin/sh", "-c"]
        args:
        - echo "Web Response" > /usr/share/nginx/html/index.html; nginx -g "daemon off;"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f multi-service.yaml

# Create Ingress
cat > ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: lab.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

kubectl apply -f ingress.yaml

# Get ingress IP
kubectl get ingress main-ingress

# Add to /etc/hosts (locally)
# <INGRESS_IP> lab.example.com

# Test routing
curl http://lab.example.com/api
curl http://lab.example.com/web
curl http://lab.example.com/
```

---

## Knowledge Check

### Quiz Questions

**Q1: What is the default Service type in Kubernetes?**
<details>
<summary>Click for Answer</summary>

**Answer**: ClusterIP - it only provides internal cluster communication.
</details>

**Q2: What port range is used for NodePort services?**
<details>
<summary>Click for Answer</summary>

**Answer**: 30000-32767 (unless configured otherwise).
</details>

**Q3: What's the difference between a normal Service and a headless Service?**
<details>
<summary>Click for Answer</summary>

**Answer**: 
- Normal Service: Has a ClusterIP, DNS returns single VIP, kube-proxy load balances
- Headless Service: No ClusterIP (`clusterIP: None`), DNS returns all pod IPs, client chooses which pod
</details>

**Q4: How does Kubernetes service discovery work?**
<details>
<summary>Click for Answer</summary>

**Answer**: Through DNS (CoreDNS). Services get DNS names in format `<service>.<namespace>.svc.cluster.local`. Pods can resolve these names to service IPs.
</details>

**Q5: What component implements Service load balancing?**
<details>
<summary>Click for Answer</summary>

**Answer**: kube-proxy runs on each node and maintains iptables or IPVS rules to route traffic to backend pods.
</details>

**Q6: When would you use an Ingress instead of LoadBalancer Service?**
<details>
<summary>Click for Answer</summary>

**Answer**: 
- Multiple HTTP services behind one IP
- Need path-based or host-based routing
- Want SSL termination
- Cost optimization (one LB vs many)
- Need L7 features (rewrites, auth, rate limiting)
</details>

**Q7: What's required for Network Policies to work?**
<details>
<summary>Click for Answer</summary>

**Answer**: A CNI plugin that supports network policies (Calico, Cilium, Weave, etc.). Default Flannel doesn't support them.
</details>

**Q8: What does `endpoint` mean in Kubernetes?**
<details>
<summary>Click for Answer</summary>

**Answer**: Endpoints are the actual Pod IPs that back a Service. An Endpoint object tracks which Pods match a Service's selector.
</details>

### Practical Exercises

1. **Exercise 1**: Create a three-tier architecture (frontend, backend, database) with appropriate Service types and Network Policies.

2. **Exercise 2**: Set up an Ingress with SSL termination using self-signed certificates.

3. **Exercise 3**: Implement blue-green deployment using Services and test zero-downtime switching.

4. **Exercise 4**: Configure a headless Service with StatefulSet and verify DNS returns individual pod IPs.

5. **Exercise 5**: Create Network Policies that implement zero-trust networking (deny all by default, allow specific paths).

---

## Summary

### Key Takeaways

✅ **Services** provide stable networking for dynamic Pods  
✅ **ClusterIP** for internal communication  
✅ **NodePort** for simple external access (dev/test)  
✅ **LoadBalancer** for production external access  
✅ **Ingress** for HTTP routing and SSL termination  
✅ **Headless Services** for stateful apps and custom load balancing  
✅ **Network Policies** for security and isolation  
✅ **DNS** is built-in for service discovery  

### Commands Cheat Sheet

```bash
# Create services
kubectl expose deployment <name> --port=80 --target-port=8080
kubectl apply -f service.yaml

# View services
kubectl get svc
kubectl get endpoints
kubectl describe svc <name>

# Test connectivity
kubectl run test --image=busybox --rm -it --restart=Never -- \
  wget -O- http://<service-name>:<port>

# DNS testing
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup <service-name>

# Network policies
kubectl get networkpolicy
kubectl apply -f network-policy.yaml

# Ingress
kubectl get ingress
kubectl describe ingress <name>
```

### What's Next?

Now that you understand networking, let's manage configuration and secrets!

➡️ **Next Section**: [ConfigMaps and Secrets](./07-configmaps-secrets.md)

In the next section:
- Decouple configuration from code
- Manage sensitive data securely
- Dynamic configuration updates
- Best practices for secrets management

---

## Additional Resources

- [Kubernetes Services Official Docs](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Networking Model](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Ingress Controllers Comparison](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Network Policies Guide](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [CoreDNS Documentation](https://coredns.io/)
- [Service Mesh Comparison](https://servicemesh.es/)

---

**🎉 Congratulations!** You've mastered Kubernetes Services and Networking!

Continue to [Section 07: ConfigMaps and Secrets →](./07-configmaps-secrets.md)
