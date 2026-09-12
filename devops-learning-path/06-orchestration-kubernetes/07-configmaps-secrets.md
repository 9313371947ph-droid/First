# Section 07: ConfigMaps & Secrets - Managing Configuration Data

## 🎯 Learning Objectives
By the end of this section, you will be able to:
- Understand the purpose of ConfigMaps and Secrets
- Create ConfigMaps from literals, files, and directories
- Mount ConfigMaps as environment variables and volumes
- Store sensitive data securely using Secrets
- Implement secret rotation strategies
- Use Kustomize for environment-specific configurations
- Follow security best practices for secrets management

## 📋 Table of Contents
1. [Why Separate Configuration from Code?](#why-separate-configuration-from-code)
2. [ConfigMaps Deep Dive](#configmaps-deep-dive)
3. [Secrets Management](#secrets-management)
4. [Using ConfigMaps and Secrets in Pods](#using-configmaps-and-secrets-in-pods)
5. [Advanced Patterns](#advanced-patterns)
6. [Security Best Practices](#security-best-practices)
7. [Troubleshooting](#troubleshooting)
8. [Hands-on Lab](#hands-on-lab)

---

## Why Separate Configuration from Code?

### The Twelve-Factor App Principle

The **Twelve-Factor App** methodology emphasizes separating configuration from code:

> "Strictly separate config from code. Config varies substantially across deploys, code does not."

**Benefits:**
- ✅ **Portability**: Same container image works across environments
- ✅ **Security**: Sensitive data not hardcoded in applications
- ✅ **Flexibility**: Change behavior without rebuilding images
- ✅ **Version Control**: Configuration changes tracked separately
- ✅ **Compliance**: Easier to audit and manage access to secrets

### Common Configuration Examples

```yaml
# Application settings
DATABASE_URL=postgres://localhost:5432/mydb
LOG_LEVEL=INFO
MAX_CONNECTIONS=100
CACHE_TTL=3600

# Feature flags
ENABLE_NEW_UI=true
BETA_FEATURES=false

# External service endpoints
API_GATEWAY_URL=https://api.example.com
SMTP_SERVER=smtp.gmail.com:587
```

---

## ConfigMaps Deep Dive

### What is a ConfigMap?

A **ConfigMap** is a Kubernetes API object that stores non-confidential data in key-value pairs.

### Creating ConfigMaps

#### Method 1: From Literals

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=db.example.com \
  --from-literal=DATABASE_PORT=5432 \
  --from-literal=LOG_LEVEL=INFO
```

#### Method 2: From a File

```bash
# Create a configuration file
cat > app.properties << EOF
database.host=localhost
database.port=5432
app.name=my-application
EOF

# Create ConfigMap from file
kubectl create configmap app-config --from-file=app.properties
```

#### Method 3: From a Directory

```bash
# Create a directory with multiple config files
mkdir config-files
echo "key1=value1" > config-files/file1.conf
echo "key2=value2" > config-files/file2.conf

# Create ConfigMap from entire directory
kubectl create configmap app-config --from-file=config-files/
```

#### Method 4: From YAML Manifest (Recommended)

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
  labels:
    app: my-app
data:
  # Simple key-value pairs
  DATABASE_HOST: "db.example.com"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "INFO"
  
  # Multi-line configuration
  app.properties: |
    database.host=localhost
    database.port=5432
    database.name=myapp
    cache.enabled=true
    cache.ttl=3600
  
  # JSON configuration
  config.json: |
    {
      "api": {
        "version": "v1",
        "timeout": 30
      },
      "features": {
        "newUI": true,
        "betaFeatures": false
      }
    }
```

Apply the ConfigMap:
```bash
kubectl apply -f configmap.yaml
```

### Viewing ConfigMaps

```bash
# List all ConfigMaps
kubectl get configmaps

# Get detailed information
kubectl describe configmap app-config

# View the full YAML
kubectl get configmap app-config -o yaml

# Get specific key value
kubectl get configmap app-config -o jsonpath='{.data.DATABASE_HOST}'
```

### Updating ConfigMaps

```bash
# Edit interactively
kubectl edit configmap app-config

# Replace entirely
kubectl create configmap app-config --from-literal=KEY=value --dry-run=client -o yaml | kubectl replace -f -

# Patch specific keys
kubectl patch configmap app-config -p '{"data":{"NEW_KEY":"new_value"}}'
```

---

## Secrets Management

### What is a Secret?

A **Secret** is similar to a ConfigMap but designed to hold sensitive data. Secrets are:
- **Base64 encoded** (not encrypted by default!)
- Stored in **tmpfs** (in-memory) on nodes
- Can be encrypted at rest (when configured)

### Types of Secrets

Kubernetes supports several built-in types:

| Type | Description |
|------|-------------|
| `Opaque` | Arbitrary user-defined data (default) |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/basic-auth` | Basic authentication credentials |
| `kubernetes.io/ssh-auth` | SSH authentication credentials |
| `kubernetes.io/token` | Service account token |

### Creating Secrets

#### Method 1: From Literals

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cr3tP@ssw0rd!'
```

⚠️ **Warning**: Command-line history may expose secrets!

#### Method 2: From Files

```bash
# Create files with sensitive data
echo -n 'admin' > username.txt
echo -n 'S3cr3tP@ssw0rd!' > password.txt

# Create Secret from files
kubectl create secret generic db-credentials \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Clean up sensitive files
shred -u username.txt password.txt
```

#### Method 3: From YAML Manifest (Recommended)

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
  labels:
    app: my-app
type: Opaque
stringData:  # Automatically base64 encodes
  username: admin
  password: S3cr3tP@ssw0rd!
  api-key: sk-1234567890abcdef
```

Or with base64-encoded data:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:  # Must be base64 encoded
  username: YWRtaW4=  # echo -n 'admin' | base64
  password: UzNjcjN0UEBzc3cwcmQh  # echo -n 'S3cr3tP@ssw0rd!' | base64
```

Apply the Secret:
```bash
kubectl apply -f secret.yaml
```

#### Method 4: From Docker Registry Credentials

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com
```

### Viewing Secrets

```bash
# List all Secrets
kubectl get secrets

# Get detailed information (values are base64 encoded)
kubectl describe secret db-credentials

# Decode and view secret value
kubectl get secret db-credentials -o jsonpath='{.data.username}' | base64 --decode

# View all decoded values
kubectl get secret db-credentials -o jsonpath='{.data}' | jq -r 'to_entries[] | "\(.key): \(.value | @base64d)"'
```

⚠️ **Security Note**: Anyone with `get` access to Secrets can decode them!

### Updating Secrets

```bash
# Edit Secret (values will be base64 encoded automatically)
kubectl edit secret db-credentials

# Patch specific keys
kubectl patch secret db-credentials -p '{"stringData":{"password":"NewP@ssw0rd!"}}'

# Delete and recreate
kubectl delete secret db-credentials
kubectl create secret generic db-credentials --from-literal=password=newpassword
```

---

## Using ConfigMaps and Secrets in Pods

### As Environment Variables

#### Using ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-configmap
spec:
  containers:
  - name: my-app
    image: my-app:latest
    env:
      # Single key from ConfigMap
      - name: DATABASE_HOST
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: DATABASE_HOST
      
      # All keys from ConfigMap
      - name: LOG_LEVEL
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: LOG_LEVEL
    
    envFrom:
      # Import all ConfigMap keys as environment variables
      - configMapRef:
          name: app-config
```

#### Using Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret
spec:
  containers:
  - name: my-app
    image: my-app:latest
    env:
      # Single key from Secret
      - name: DB_USERNAME
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: username
      
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: password
    
    envFrom:
      # Import all Secret keys as environment variables
      - secretRef:
          name: db-credentials
```

### As Volume Mounts

#### Mounting ConfigMap as Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-config-volume
spec:
  containers:
  - name: my-app
    image: nginx:latest
    volumeMounts:
      # Mount entire ConfigMap
      - name: config-volume
        mountPath: /etc/config
      
      # Mount specific key to specific path
      - name: config-volume
        mountPath: /etc/app/app.properties
        subPath: app.properties
      
      # Mount as read-only
      - name: config-volume
        mountPath: /etc/config-ro
        readOnly: true
  
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        # Optional: set permissions
        defaultMode: 0644
        # Optional: only include specific keys
        items:
          - key: app.properties
            path: app.properties
          - key: config.json
            path: config.json
```

#### Mounting Secret as Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-secret-volume
spec:
  containers:
  - name: my-app
    image: nginx:latest
    volumeMounts:
      - name: secret-volume
        mountPath: /etc/secrets
        readOnly: true
  
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
        defaultMode: 0400  # Read-only for owner
        # Optional: only include specific keys
        items:
          - key: username
            path: db-username
          - key: password
            path: db-password
```

### Complete Example: Web Application

```yaml
# complete-app.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_ENV: production
  LOG_LEVEL: INFO
  MAX_CONNECTIONS: "100"
  CACHE_TTL: "3600"
---
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
stringData:
  DATABASE_URL: postgres://user:password@db:5432/webapp
  API_KEY: sk-1234567890abcdef
  JWT_SECRET: super-secret-jwt-key
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
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
      - name: webapp
        image: my-webapp:latest
        ports:
          - containerPort: 8080
        
        # Environment variables from ConfigMap
        env:
          - name: APP_ENV
            valueFrom:
              configMapKeyRef:
                name: webapp-config
                key: APP_ENV
          
          - name: DATABASE_URL
            valueFrom:
              secretKeyRef:
                name: webapp-secret
                key: DATABASE_URL
        
        # Mount configuration files
        volumeMounts:
          - name: config-volume
            mountPath: /app/config
            readOnly: true
          - name: secret-volume
            mountPath: /app/secrets
            readOnly: true
      
      volumes:
        - name: config-volume
          configMap:
            name: webapp-config
        - name: secret-volume
          secret:
            secretName: webapp-secret
```

---

## Advanced Patterns

### Hot Reload Configuration

ConfigMaps mounted as volumes can be updated without restarting pods:

```bash
# Update ConfigMap
kubectl create configmap app-config --from-literal=LOG_LEVEL=DEBUG --dry-run=client -o yaml | kubectl replace -f -

# Pod will see the change within 1-2 minutes
# Note: Application must support hot-reload
```

⚠️ **Limitations**:
- Updates take 1-2 minutes to propagate
- Applications must reload config files
- Environment variables don't update automatically

### Using initContainers for Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  containers:
  - name: my-app
    image: my-app:latest
    envFrom:
      - configMapRef:
          name: app-config-final
  
  initContainers:
  - name: prepare-config
    image: busybox
    command: ['sh', '-c']
    args:
      - |
        echo "Preparing configuration..."
        # Validate config
        if [ -z "$DATABASE_HOST" ]; then
          echo "ERROR: DATABASE_HOST not set"
          exit 1
        fi
        # Transform or merge configs
        cp /config/* /final-config/
    envFrom:
      - configMapRef:
          name: app-config-source
    volumeMounts:
      - name: final-config
        mountPath: /final-config
      - name: config
        mountPath: /config
  
  volumes:
    - name: final-config
      emptyDir: {}
    - name: config
      configMap:
        name: app-config-source
```

### Secret Rotation Strategy

#### Manual Rotation

```bash
# 1. Create new secret with updated credentials
kubectl create secret generic db-credentials-v2 \
  --from-literal=username=admin \
  --from-literal=password='NewP@ssw0rd!'

# 2. Update deployment to use new secret
kubectl set env deployment/myapp --from=secret/db-credentials-v2

# 3. Trigger rolling update
kubectl rollout restart deployment/myapp

# 4. Verify and clean up old secret
kubectl delete secret db-credentials
```

#### Automated Rotation with External Secrets

```yaml
# Using External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials-k8s
  data:
    - secretKey: username
      remoteRef:
        key: prod/database
        property: username
    - secretKey: password
      remoteRef:
        key: prod/database
        property: password
```

### Using Kustomize for Environment-Specific Configs

```bash
# Directory structure
config/
├── base/
│   ├── kustomization.yaml
│   ├── configmap.yaml
│   └── secret.yaml
├── overlays/
│   ├── development/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - configmap.yaml
  - secret.yaml

# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=WARN
      - MAX_CONNECTIONS=500
secretGenerator:
  - name: db-credentials
    behavior: merge
    literals:
      - password=ProductionSecretPassword!
```

Apply with Kustomize:
```bash
kubectl apply -k overlays/production/
```

---

## Security Best Practices

### ✅ DO:

1. **Use RBAC to restrict access**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["allowed-secret"]  # Only specific secrets
```

2. **Enable encryption at rest**
```yaml
# In kube-apiserver configuration
--encryption-provider-config=/path/to/encryption-config.yaml
```

3. **Use external secret management**
   - AWS Secrets Manager
   - Azure Key Vault
   - HashiCorp Vault
   - Google Secret Manager

4. **Implement secret rotation policies**
   - Rotate secrets regularly (every 90 days)
   - Automate rotation when possible
   - Monitor for leaked secrets

5. **Audit secret access**
```bash
kubectl audit-policy.yaml
```

### ❌ DON'T:

1. **Don't commit secrets to Git**
   ```bash
   # Add to .gitignore
   *.secret.yaml
   *credentials*
   ```

2. **Don't use Secrets as environment variables unnecessarily**
   - Prefer volume mounts (more secure)
   - Environment variables can be exposed in process listings

3. **Don't rely solely on base64 encoding**
   - Base64 is NOT encryption
   - Anyone with access can decode

4. **Don't share Secrets across namespaces**
   - Secrets are namespace-scoped
   - Use separate secrets per namespace

5. **Don't log sensitive data**
   ```yaml
   # Bad: Logging passwords
   kubectl logs my-pod | grep password
   ```

---

## Troubleshooting

### Common Issues

#### ConfigMap Not Found

```bash
# Check if ConfigMap exists
kubectl get configmap app-config

# Verify namespace
kubectl get configmap app-config -n <namespace>

# Check pod events
kubectl describe pod my-pod
```

#### Secret Mount Permissions

```bash
# Check volume mount permissions
kubectl exec my-pod -- ls -la /etc/secrets

# Fix: Adjust defaultMode in volume definition
volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
      defaultMode: 0400  # Read-only for owner
```

#### Environment Variables Not Set

```bash
# Check if key exists in ConfigMap/Secret
kubectl get configmap app-config -o jsonpath='{.data.KEY_NAME}'

# Verify pod spec references correct keys
kubectl get pod my-pod -o yaml | grep -A 10 env
```

#### Config Changes Not Applied

```bash
# For volume mounts, changes propagate automatically (1-2 min delay)
# For environment variables, you must restart the pod

# Force restart
kubectl rollout restart deployment/myapp

# Verify ConfigMap was updated
kubectl get configmap app-config -o yaml
```

### Debug Commands

```bash
# List all ConfigMaps in namespace
kubectl get configmaps -n <namespace>

# List all Secrets in namespace
kubectl get secrets -n <namespace>

# Export ConfigMap to file
kubectl get configmap app-config -o yaml > configmap-backup.yaml

# Export Secret to file (be careful!)
kubectl get secret db-credentials -o yaml > secret-backup.yaml

# Compare ConfigMaps
kubectl get configmap app-config-v1 -o yaml > v1.yaml
kubectl get configmap app-config-v2 -o yaml > v2.yaml
diff v1.yaml v2.yaml

# Test ConfigMap in a temporary pod
kubectl run test-config --rm -it --image=busybox --restart=Never \
  --env-from=configmap/app-config -- sh
```

---

## Hands-on Lab

### Lab 7: Configure an Application with ConfigMaps and Secrets

#### Scenario
You need to deploy a web application that requires:
- Database connection strings (secret)
- Application configuration (configmap)
- API keys (secret)
- Environment-specific settings

#### Prerequisites
- Kubernetes cluster (from Section 03)
- kubectl configured
- Basic understanding of Deployments (Section 05)

#### Tasks

**Task 1: Create ConfigMaps**

```bash
# Create a ConfigMap for application settings
kubectl create configmap webapp-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=MAX_CONNECTIONS=100 \
  --from-literal=CACHE_TTL=3600

# Verify
kubectl get configmap webapp-config -o yaml
```

**Task 2: Create Secrets**

```bash
# Create a Secret for database credentials
kubectl create secret generic db-credentials \
  --from-literal=username=webapp_user \
  --from-literal=password='Str0ng_P@ssw0rd!' \
  --from-literal=host=db.example.com \
  --from-literal=port=5432

# Create a Secret for API keys
kubectl create secret generic api-keys \
  --from-literal=stripe-key=sk_test_abc123 \
  --from-literal=sendgrid-key=SG.xyz789

# Verify (decode values)
kubectl get secret db-credentials -o jsonpath='{.data}' | jq -r 'to_entries[] | "\(.key): \(.value | @base64d)"'
```

**Task 3: Create Deployment**

Create a file `webapp-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:latest
        ports:
          - containerPort: 80
        
        # Environment variables from ConfigMap
        env:
          - name: APP_ENV
            valueFrom:
              configMapKeyRef:
                name: webapp-config
                key: APP_ENV
          
          - name: LOG_LEVEL
            valueFrom:
              configMapKeyRef:
                name: webapp-config
                key: LOG_LEVEL
          
          # Sensitive data from Secrets
          - name: DB_USERNAME
            valueFrom:
              secretKeyRef:
                name: db-credentials
                key: username
          
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: db-credentials
                key: password
        
        # Mount configuration files
        volumeMounts:
          - name: config-volume
            mountPath: /etc/app/config
          - name: secret-volume
            mountPath: /etc/app/secrets
            readOnly: true
      
      volumes:
        - name: config-volume
          configMap:
            name: webapp-config
        - name: secret-volume
          secret:
            secretName: api-keys
```

Deploy:
```bash
kubectl apply -f webapp-deployment.yaml
```

**Task 4: Verify Configuration**

```bash
# Check pods are running
kubectl get pods -l app=webapp

# Exec into a pod and verify environment variables
kubectl exec -it <pod-name> -- env | grep -E '(APP_ENV|LOG_LEVEL|DB_)'

# Check mounted files
kubectl exec -it <pod-name> -- ls -la /etc/app/config
kubectl exec -it <pod-name> -- cat /etc/app/config/LOG_LEVEL

# Check secrets are mounted
kubectl exec -it <pod-name> -- ls -la /etc/app/secrets
kubectl exec -it <pod-name> -- cat /etc/app/secrets/stripe-key
```

**Task 5: Update Configuration**

```bash
# Update ConfigMap
kubectl create configmap webapp-config \
  --from-literal=APP_ENV=staging \
  --from-literal=LOG_LEVEL=DEBUG \
  --from-literal=MAX_CONNECTIONS=200 \
  --from-literal=CACHE_TTL=1800 \
  --dry-run=client -o yaml | kubectl replace -f -

# Watch the ConfigMap change
watch kubectl get configmap webapp-config

# Restart deployment to pick up environment variable changes
kubectl rollout restart deployment/webapp

# Monitor rollout
kubectl rollout status deployment/webapp
```

**Task 6: Create Environment-Specific Overlays**

Create development overlay:
```bash
mkdir -p overlays/development
cat > overlays/development/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
configMapGenerator:
  - name: webapp-config
    behavior: merge
    literals:
      - APP_ENV=development
      - LOG_LEVEL=DEBUG
secretGenerator:
  - name: db-credentials
    behavior: merge
    literals:
      - password=DevPassword123
EOF

# Apply development configuration
kubectl apply -k overlays/development/
```

#### Success Criteria
✅ ConfigMap created with application settings
✅ Secrets created and properly encoded
✅ Deployment uses both ConfigMaps and Secrets
✅ Environment variables correctly set in pods
✅ Configuration files mounted as volumes
✅ Configuration updates applied successfully
✅ Environment-specific overrides working

---

## Knowledge Check

### Quiz Questions

1. **What is the main difference between ConfigMaps and Secrets?**
   - A) ConfigMaps are encrypted, Secrets are not
   - B) Secrets are designed for sensitive data and have additional security features
   - C) ConfigMaps can only store strings, Secrets can store binary data
   - D) There is no difference

2. **How are Secret values stored in Kubernetes?**
   - A) Encrypted with AES-256
   - B) Plain text in etcd
   - C) Base64 encoded
   - D) Hashed with SHA-256

3. **Which method is NOT a way to create a ConfigMap?**
   - A) From literal values
   - B) From a file
   - C) From a Docker image
   - D) From a YAML manifest

4. **What happens when you update a ConfigMap mounted as a volume?**
   - A) Pods must be restarted
   - B) Changes propagate automatically within 1-2 minutes
   - C) Changes never propagate
   - D) A new pod is created

5. **Best practice for managing secrets in production:**
   - A) Store in Git with encryption
   - B) Use environment variables only
   - C) Use external secret management (Vault, AWS Secrets Manager)
   - D) Hardcode in application

<details>
<summary><strong>Click to reveal answers</strong></summary>

1. **B** - Secrets are designed for sensitive data and have additional security features
2. **C** - Base64 encoded (but can be encrypted at rest with proper configuration)
3. **C** - From a Docker image
4. **B** - Changes propagate automatically within 1-2 minutes (for volume mounts)
5. **C** - Use external secret management (Vault, AWS Secrets Manager)

</details>

---

## Summary

### Key Takeaways

✅ **ConfigMaps** store non-sensitive configuration data as key-value pairs
✅ **Secrets** store sensitive data with base64 encoding and additional security features
✅ Both can be used as **environment variables** or **volume mounts**
✅ **Volume mounts** support hot-reload; environment variables require pod restart
✅ Always use **RBAC** to restrict access to Secrets
✅ Consider **external secret management** for production workloads
✅ Use **Kustomize** or **Helm** for environment-specific configurations

### What's Next?

Now that you can manage configuration data, let's explore **persistent storage** in Kubernetes:

➡️ **Next Section**: [Volumes & Persistent Storage](./08-volumes-storage.md)

Learn how to:
- Persist data beyond pod lifecycle
- Use different volume types (emptyDir, hostPath, PVC)
- Implement dynamic provisioning
- Work with StatefulSets and databases

---

## Additional Resources

- [Official Documentation: ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Official Documentation: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [External Secrets Operator](https://external-secrets.io/)
- [HashiCorp Vault with Kubernetes](https://www.vaultproject.io/docs/platform/k8s)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)

---

**Ready for the next challenge?** Continue to [Section 08: Volumes & Persistent Storage](./08-volumes-storage.md)
