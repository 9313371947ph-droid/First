# Section 08: Volumes & Persistent Storage in Kubernetes

## 🎯 Learning Objectives
By the end of this section, you will be able to:
- Understand why persistent storage is needed in Kubernetes
- Work with different volume types (emptyDir, hostPath, NFS, etc.)
- Create and manage PersistentVolumes (PV) and PersistentVolumeClaims (PVC)
- Implement dynamic provisioning with StorageClasses
- Configure StatefulSets for stateful applications
- Backup and restore persistent data
- Troubleshoot common storage issues

## 📋 Table of Contents
1. [The Storage Challenge in Kubernetes](#the-storage-challenge-in-kubernetes)
2. [Volume Types Overview](#volume-types-overview)
3. [PersistentVolumes (PV)](#persistentvolumes-pv)
4. [PersistentVolumeClaims (PVC)](#persistentvolumeclaims-pvc)
5. [StorageClasses & Dynamic Provisioning](#storageclasses--dynamic-provisioning)
6. [StatefulSets for Stateful Applications](#statefulsets-for-stateful-applications)
7. [Backup & Restore Strategies](#backup--restore-strategies)
8. [Troubleshooting Storage Issues](#troubleshooting-storage-issues)
9. [Hands-on Lab](#hands-on-lab)

---

## The Storage Challenge in Kubernetes

### Why is Storage Difficult in Kubernetes?

**Pods are ephemeral**: When a pod dies, all data inside it is lost forever.

```bash
# Pod with local data
kubectl run myapp --image=myapp:latest
kubectl exec myapp -- touch /data/important.txt

# Pod crashes or gets rescheduled
kubectl delete pod myapp

# New pod created - data is gone!
kubectl run myapp --image=myapp:latest
kubectl exec myapp -- ls /data
# Output: empty!
```

### Scenarios Requiring Persistent Storage

| Scenario | Requirement |
|----------|-------------|
| **Databases** | MySQL, PostgreSQL, MongoDB need persistent data |
| **File Uploads** | User-uploaded content must persist |
| **Logs & Monitoring** | Centralized log storage |
| **Shared Data** | Multiple pods accessing same files |
| **Stateful Applications** | Redis, Elasticsearch, Kafka |

---

## Volume Types Overview

Kubernetes supports many volume types:

### Basic Volume Types

#### 1. emptyDir

Temporary storage that exists only while the pod runs.

```yaml
volumes:
  - name: cache-volume
    emptyDir: {}
```

**Use cases:**
- Scratch space for computations
- Sharing data between containers in same pod
- Temporary cache

**Limitations:**
- ❌ Data deleted when pod is deleted
- ❌ Not persistent across restarts

#### 2. hostPath

Mounts a file/directory from the host node.

```yaml
volumes:
  - name: log-volume
    hostPath:
      path: /var/log/app
      type: DirectoryOrCreate
```

**Use cases:**
- Accessing system logs
- Running privileged containers (e.g., monitoring agents)

**Limitations:**
- ❌ Tied to specific node
- ❌ Security risks
- ❌ Not portable

#### 3. configMap & secret

Already covered in Section 07 - used for configuration data.

### Network Volume Types

#### 4. nfs (Network File System)

```yaml
volumes:
  - name: nfs-volume
    nfs:
      server: nfs-server.example.com
      path: /exports/data
```

#### 5. awsElasticBlockStore (AWS EBS)

```yaml
volumes:
  - name: aws-volume
    awsElasticBlockStore:
      volumeID: vol-0abcd1234efgh5678
      fsType: ext4
```

#### 6. azureDisk & azureFile

```yaml
volumes:
  - name: azure-volume
    azureDisk:
      diskName: mydisk
      diskURI: /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Compute/disks/mydisk
```

#### 7. gcePersistentDisk (GCP)

```yaml
volumes:
  - name: gce-volume
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```

#### 8. cephfs, glusterfs, vsphereVolume, etc.

Many other storage backends supported!

---

## PersistentVolumes (PV)

### What is a PersistentVolume?

A **PersistentVolume (PV)** is a cluster-wide storage resource provisioned by an administrator.

### PV Lifecycle

```
Provisioning → Available → Bound → Released → Failed
```

### Creating a PersistentVolume

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
  labels:
    type: local
spec:
  # Storage capacity
  capacity:
    storage: 10Gi
  
  # Access modes
  accessModes:
    - ReadWriteOnce  # Can be mounted as read-write by a single node
  
  # Reclaim policy (what happens after PVC is deleted)
  persistentVolumeReclaimPolicy: Retain  # Options: Retain, Recycle, Delete
  
  # Storage class
  storageClassName: manual
  
  # Volume type (NFS example)
  nfs:
    path: /data
    server: nfs-server.example.com
  
  # Mount options
  mountOptions:
    - hard
    - nfsvers=4.1
  
  # Node affinity (optional - tie to specific nodes)
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node-1
                - node-2
```

Apply the PV:
```bash
kubectl apply -f pv.yaml
```

### Access Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **ReadWriteOnce (RWO)** | Read-write by single node | Most databases |
| **ReadOnlyMany (ROX)** | Read-only by multiple nodes | Static content, shared configs |
| **ReadWriteMany (RWX)** | Read-write by multiple nodes | Shared uploads, collaborative apps |

### Reclaim Policies

| Policy | Behavior |
|--------|----------|
| **Retain** | PV kept after PVC deletion (manual cleanup required) |
| **Recycle** | Data scrubbed, PV made available again (deprecated) |
| **Delete** | PV and underlying storage deleted (use with caution!) |

### Viewing PVs

```bash
# List all PVs
kubectl get pv

# Detailed information
kubectl describe pv pv-data

# Filter by status
kubectl get pv --field-selector status.phase=Available
```

---

## PersistentVolumeClaims (PVC)

### What is a PersistentVolumeClaim?

A **PersistentVolumeClaim (PVC)** is a request for storage by a user/pod.

Think of it like this:
- **PV** = Storage resource (like a physical disk)
- **PVC** = Request for storage (like a partition request)
- **Pod** = Uses the PVC (like an application using the partition)

### Creating a PVC

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
  namespace: default
spec:
  # Requested access mode
  accessModes:
    - ReadWriteOnce
  
  # Requested storage class
  storageClassName: manual
  
  # Requested size
  resources:
    requests:
      storage: 5Gi
  
  # Optional: Select specific PV
  # selector:
  #   matchLabels:
  #     type: local
```

Apply the PVC:
```bash
kubectl apply -f pvc.yaml
```

### PVC Binding Process

Kubernetes automatically binds PVC to matching PV based on:
1. ✅ Access modes compatibility
2. ✅ Storage class match
3. ✅ Sufficient capacity
4. ✅ Selector labels (if specified)

```bash
# Check binding status
kubectl get pvc
# NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# pvc-data   Bound    pv-data   10Gi       RWO            manual         5m

# If status is Pending, check why
kubectl describe pvc pvc-data
```

### Using PVC in a Pod

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
    - name: my-app
      image: nginx:latest
      ports:
        - containerPort: 80
      
      # Mount the PVC
      volumeMounts:
        - name: data-volume
          mountPath: /usr/share/nginx/html
          subPath: html  # Optional: mount specific subdirectory
        
        - name: data-volume
          mountPath: /var/log/app
          subPath: logs
  
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: pvc-data
        # Optional: declare read-only
        # readOnly: true
```

### Complete Example: Database with Persistent Storage

```yaml
# mysql-persistent.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-mysql-data
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/data/mysql
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-mysql-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
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
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: pvc-mysql-data
```

---

## StorageClasses & Dynamic Provisioning

### What is a StorageClass?

A **StorageClass** defines how storage should be provisioned automatically.

**Benefits:**
- ✅ No need to pre-provision PVs
- ✅ Automatic provisioning when PVC is created
- ✅ Different tiers of storage (fast, slow, ssd, hdd)
- ✅ Cloud provider integration

### Creating a StorageClass

#### AWS EBS Example

```yaml
# storageclass-aws.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3  # gp2, gp3, io1, io2, st1, sc1
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
mountOptions:
  - debug
allowVolumeExpansion: true
```

#### GCP PD Example

```yaml
# storageclass-gcp.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: none
reclaimPolicy: Delete
allowVolumeExpansion: true
```

#### Azure Disk Example

```yaml
# storageclass-azure.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Premium_LRS
  kind: Managed
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

#### NFS with Dynamic Provisioning

```yaml
# storageclass-nfs.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "false"
```

### Using StorageClass in PVC

```yaml
# pvc-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic-storage
spec:
  storageClassName: fast-ssd  # Reference StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

When you apply this PVC:
1. Kubernetes sees `storageClassName: fast-ssd`
2. Contacts the provisioner (`kubernetes.io/aws-ebs`)
3. Creates EBS volume automatically
4. Creates PV bound to the PVC

```bash
# Watch dynamic provisioning
kubectl get pvc -w
# NAME                STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# pvc-dynamic-storage Pending                                      fast-ssd       0s
# pvc-dynamic-storage Bound      pvc-abc123   10Gi       RWO            fast-ssd       5s
```

### Default StorageClass

Set a StorageClass as default:

```bash
# Mark as default
kubectl patch storageclass fast-ssd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
# NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
# fast-ssd     kubernetes.io/aws-ebs   Delete          Immediate           true                   10m (default)
```

Now PVCs without explicit `storageClassName` use the default.

---

## StatefulSets for Stateful Applications

### What are StatefulSets?

**StatefulSets** manage stateful applications with:
- ✅ Stable, unique network identifiers
- ✅ Stable, persistent storage
- ✅ Ordered, graceful deployment/scaling
- ✅ Ordered, automated rolling updates

### StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random (web-abc123) | Predictable (web-0, web-1, web-2) |
| Storage | Shared (same PVC) | Unique (one PVC per pod) |
| Ordering | No guarantee | Ordered deployment & termination |
| Use case | Stateless apps | Stateful apps (databases, clusters) |

### Creating a StatefulSet

```yaml
# statefulset-mysql.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-cluster
spec:
  serviceName: mysql  # Headless service name
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
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: rootpassword
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  
  # Volume claim templates (creates one PVC per pod)
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 10Gi
```

### Headless Service for StatefulSet

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
      name: mysql
  clusterIP: None  # Headless service
  selector:
    app: mysql
```

### StatefulSet Pod Naming

```bash
# Pods have predictable names
kubectl get pods
# NAME            READY   STATUS    RESTARTS   AGE
# mysql-cluster-0   1/1     Running   0          5m
# mysql-cluster-1   1/1     Running   0          4m
# mysql-cluster-2   1/1     Running   0          3m

# Each has its own DNS entry
# mysql-cluster-0.mysql.default.svc.cluster.local
# mysql-cluster-1.mysql.default.svc.cluster.local
# mysql-cluster-2.mysql.default.svc.cluster.local

# Each has its own PVC
kubectl get pvc
# NAME                      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# data-mysql-cluster-0      Bound    pv-001   10Gi       RWO            fast-ssd       5m
# data-mysql-cluster-1      Bound    pv-002   10Gi       RWO            fast-ssd       4m
# data-mysql-cluster-2      Bound    pv-003   10Gi       RWO            fast-ssd       3m
```

### Scaling StatefulSets

```bash
# Scale up (adds mysql-cluster-3)
kubectl scale statefulset mysql-cluster --replicas=4

# Scale down (removes mysql-cluster-3 first)
kubectl scale statefulset mysql-cluster --replicas=3
```

### Rolling Updates

```bash
# Update image
kubectl set image statefulset/mysql-cluster mysql=mysql:8.1

# Monitor rollout
kubectl rollout status statefulset/mysql-cluster

# Pods update in order (0 → 1 → 2)
```

---

## Backup & Restore Strategies

### Backup Methods

#### 1. Volume Snapshots

```yaml
# volumesnapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-backup-snapshot
spec:
  volumeSnapshotClassName: csi-aws-vol-snap-class
  source:
    persistentVolumeClaimName: pvc-mysql-data
```

Restore from snapshot:
```yaml
# pvc-from-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-mysql-restored
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  dataSource:
    name: mysql-backup-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  resources:
    requests:
      storage: 20Gi
```

#### 2. Database Native Backup

```bash
# MySQL backup
kubectl exec mysql-cluster-0 -- mysqldump -u root -p'password' --all-databases > backup.sql

# PostgreSQL backup
kubectl exec postgres-0 -- pg_dumpall -U postgres > backup.sql

# MongoDB backup
kubectl exec mongo-0 -- mongodump --out=/backup
```

#### 3. rsync Backup

```bash
# Backup volume to local machine
kubectl run backup-pod --rm -it --image=alpine \
  --volume <pvc-name>:<mount-path> \
  -- sh -c "tar czf - /data" > backup.tar.gz
```

#### 4. Velero (Recommended for Production)

[Velero](https://velero.io/) is a backup tool for Kubernetes:

```bash
# Install Velero
velero install \
  --provider aws \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --use-volume-snapshots=true

# Backup entire namespace
velero backup create mysql-backup --include-namespaces default

# Restore
velero restore create --from-backup mysql-backup
```

### Disaster Recovery Plan

1. **Regular backups** (daily/hourly depending on RPO)
2. **Test restores** monthly
3. **Off-site copies** (different region/cloud)
4. **Documented procedures** for recovery
5. **Automated verification** of backup integrity

---

## Troubleshooting Storage Issues

### Common Problems

#### PVC Stuck in Pending

```bash
# Check PVC events
kubectl describe pvc pvc-name

# Common causes:
# 1. No matching PV available
# 2. StorageClass doesn't exist
# 3. Insufficient capacity
# 4. Access mode incompatibility

# Solutions:
# - Create matching PV
# - Fix StorageClass name
# - Increase PV capacity
# - Change access mode
```

#### PV Stuck in Terminating

```bash
# Check if finalizer is blocking
kubectl get pv pv-name -o yaml | grep finalizers

# Remove finalizer (careful!)
kubectl patch pv pv-name -p '{"metadata":{"finalizers":null}}'
```

#### Mount Errors

```bash
# Check pod events
kubectl describe pod pod-name

# Common errors:
# - Volume not available on node
# - Permission denied
# - Wrong filesystem type

# Solutions:
# - Use appropriate access mode
# - Check node affinity
# - Verify fsType matches
```

### Debug Commands

```bash
# List all storage resources
kubectl get pv,pvc,storageclass

# Check which pods use a PVC
kubectl get pods --all-namespaces -o jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.spec.volumes[*].persistentVolumeClaim.claimName}{'\n'}{end}"

# Test volume access
kubectl run test-pod --rm -it --image=busybox \
  --overrides='{"spec":{"volumes":[{"name":"test-vol","persistentVolumeClaim":{"claimName":"pvc-name"}}],"containers":[{"name":"test","image":"busybox","command":["sh"],"stdin":true,"tty":true,"volumeMounts":[{"name":"test-vol","mountPath":"/data"}]}]}}'

# Check node storage capacity
kubectl describe node node-name | grep -A 10 "Allocated resources"
```

---

## Hands-on Lab

### Lab 8: Persistent Storage for WordPress + MySQL

#### Scenario
Deploy a WordPress site with persistent database storage that survives pod restarts.

#### Prerequisites
- Kubernetes cluster with StorageClass support
- kubectl configured

#### Tasks

**Task 1: Create Namespace**

```bash
kubectl create namespace wordpress-lab
kubectl config set-context --current --namespace=wordpress-lab
```

**Task 2: Create Secret for MySQL**

```bash
kubectl create secret generic mysql-secret \
  --from-literal=root-password='MyS3cr3tR00t!' \
  --from-literal=password='W0rdPr3ssDB!' \
  --from-literal=username=wpuser
```

**Task 3: Create PVC for MySQL**

```yaml
# mysql-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard  # Or your cluster's default
```

```bash
kubectl apply -f mysql-pvc.yaml
```

**Task 4: Deploy MySQL with StatefulSet**

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: wordpress-lab
spec:
  serviceName: mysql
  replicas: 1
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
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            - name: MYSQL_DATABASE
              value: wordpress
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: username
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress-lab
spec:
  ports:
    - port: 3306
  clusterIP: None
  selector:
    app: mysql
```

```bash
kubectl apply -f mysql-statefulset.yaml
```

**Task 5: Create PVC for WordPress**

```yaml
# wp-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

```bash
kubectl apply -f wp-pvc.yaml
```

**Task 6: Deploy WordPress**

```yaml
# wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql:3306
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: username
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            - name: WORDPRESS_DB_NAME
              value: wordpress
          volumeMounts:
            - name: wordpress-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-storage
          persistentVolumeClaim:
            claimName: wordpress-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress-lab
spec:
  type: NodePort  # Or LoadBalancer for cloud
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: wordpress
```

```bash
kubectl apply -f wordpress-deployment.yaml
```

**Task 7: Verify Persistence**

```bash
# Check all resources
kubectl get all,pvc,pv

# Access WordPress (get NodePort or LoadBalancer IP)
kubectl get svc wordpress

# Test persistence - delete pods
kubectl delete pod -l app=mysql
kubectl delete pod -l app=wordpress

# Wait for recreation
kubectl get pods -w

# Verify data persists
kubectl exec -it <mysql-pod> -- mysql -u root -p'MyS3cr3tR00t!' -e "SHOW DATABASES;"
# Should still show 'wordpress' database
```

**Task 8: Create Backup**

```bash
# Backup MySQL
MYSQL_POD=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')
kubectl exec $MYSQL_POD -- mysqldump -u root -p'MyS3cr3tR00t!' wordpress > wordpress-backup.sql

# Verify backup
head -20 wordpress-backup.sql
```

#### Success Criteria
✅ MySQL StatefulSet running with persistent storage
✅ WordPress Deployment running with persistent storage
✅ Database connection working
✅ Data survives pod deletion
✅ Backup created successfully

---

## Knowledge Check

### Quiz Questions

1. **What happens to data in an emptyDir volume when a pod is deleted?**
   - A) It persists
   - B) It's backed up automatically
   - C) It's deleted
   - D) It moves to the host

2. **Which access mode allows multiple nodes to read and write?**
   - A) ReadWriteOnce (RWO)
   - B) ReadOnlyMany (ROX)
   - C) ReadWriteMany (RWX)
   - D) ExclusiveAccess

3. **What is the purpose of a StorageClass?**
   - A) To classify PVs by size
   - B) To enable dynamic provisioning
   - C) To encrypt volumes
   - D) To backup PVCs

4. **How do StatefulSets differ from Deployments?**
   - A) They're faster
   - B) They provide stable network identities and ordered deployment
   - C) They use less memory
   - D) They don't support scaling

5. **Best practice for database backups in Kubernetes:**
   - A) Only rely on PV snapshots
   - B) Use database-native backup tools + off-site storage
   - C) Copy files manually
   - D) Don't backup, use replication instead

<details>
<summary><strong>Click to reveal answers</strong></summary>

1. **C** - It's deleted
2. **C** - ReadWriteMany (RWX)
3. **B** - To enable dynamic provisioning
4. **B** - They provide stable network identities and ordered deployment
5. **B** - Use database-native backup tools + off-site storage

</details>

---

## Summary

### Key Takeaways

✅ **Volumes** provide storage to pods; different types for different needs
✅ **PersistentVolumes (PV)** are cluster storage resources
✅ **PersistentVolumeClaims (PVC)** request storage for pods
✅ **StorageClasses** enable dynamic provisioning
✅ **StatefulSets** manage stateful applications with stable identities
✅ **Backup strategies** are critical for production workloads
✅ Choose the right **access mode** for your use case

### What's Next?

Now that you understand storage, let's explore **advanced workload controllers**:

➡️ **Next Section**: [StatefulSets & DaemonSets](./09-statefulsets-daemonsets.md)

Learn about:
- Advanced StatefulSet patterns
- DaemonSets for node-level services
- Jobs and CronJobs for batch processing
- When to use each controller type

---

## Additional Resources

- [Official Documentation: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Official Documentation: PV & PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Official Documentation: StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Official Documentation: StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Velero Backup Tool](https://velero.io/)
- [CSI Drivers](https://kubernetes-csi.github.io/docs/)

---

**Ready to continue?** Move to [Section 09: StatefulSets & DaemonSets](./09-statefulsets-daemonsets.md)
