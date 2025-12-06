# CKA Study Guide: Storage (10%)

## Domain Overview

The Storage domain covers managing persistent storage in Kubernetes clusters. This includes implementing storage classes, dynamic volume provisioning, configuring volume types, access modes, reclaim policies, and managing Persistent Volumes (PVs) and Persistent Volume Claims (PVCs).

---

## 1. Implement Storage Classes and Dynamic Volume Provisioning

### 1.1 Understanding StorageClass

A **StorageClass** provides a way for administrators to describe the classes of storage they offer. Different classes might map to quality-of-service levels, backup policies, or arbitrary policies determined by cluster administrators.

#### StorageClass Components

| Field | Description |
|-------|-------------|
| `provisioner` | Determines what volume plugin is used for provisioning PVs |
| `parameters` | Parameters passed to the provisioner |
| `reclaimPolicy` | What happens to the PV when released (Delete/Retain) |
| `volumeBindingMode` | When binding and provisioning should occur |
| `allowVolumeExpansion` | Whether the storage class allows volume expansion |
| `mountOptions` | Mount options for dynamically created PVs |

#### Example StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Set as default
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
mountOptions:
  - debug
```

### 1.2 Dynamic Volume Provisioning

Dynamic volume provisioning eliminates the need for cluster administrators to pre-provision storage. Instead, it automatically provisions storage when users create PersistentVolumeClaim objects.

#### How Dynamic Provisioning Works

1. Administrator creates StorageClass objects
2. User creates a PVC requesting a specific StorageClass
3. The provisioner automatically creates a PV matching the PVC specifications
4. The PV is bound to the PVC

#### Enabling Dynamic Provisioning

- Ensure the `DefaultStorageClass` admission controller is enabled on the API server
- Create at least one StorageClass
- Reference the StorageClass in your PVC or set a default StorageClass

```yaml
# PVC requesting dynamic provisioning
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage  # References the StorageClass
  resources:
    requests:
      storage: 10Gi
```

### 1.3 Default StorageClass

You can mark a StorageClass as default:

```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```

**Important**: If multiple StorageClasses are marked as default, Kubernetes uses the most recently created one.

---

## 2. Configure Volume Types, Access Modes, and Reclaim Policies

### 2.1 Volume Types

Kubernetes supports many volume types:

| Volume Type | Description | Use Case |
|-------------|-------------|----------|
| `emptyDir` | Temporary directory, deleted when Pod terminates | Scratch space, caching |
| `hostPath` | Mounts directory from the host node | Node-level storage, testing |
| `nfs` | Network File System mount | Shared storage across Pods |
| `configMap` | Provides config data to Pods | Configuration files |
| `secret` | Provides sensitive data to Pods | Passwords, tokens, keys |
| `persistentVolumeClaim` | Claims a PersistentVolume | Persistent application data |
| `csi` | Container Storage Interface volumes | Cloud provider storage |
| `projected` | Projects multiple volume sources | Combined secrets/configmaps |

#### EmptyDir Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: container
    image: nginx
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi  # Optional size limit
```

#### HostPath Example (Use with caution)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: container
    image: nginx
    volumeMounts:
    - mountPath: /host-data
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: /data
      type: DirectoryOrCreate  # Types: Directory, DirectoryOrCreate, File, FileOrCreate
```

### 2.2 Access Modes

Access modes define how a volume can be mounted:

| Access Mode | Abbreviation | Description |
|-------------|--------------|-------------|
| `ReadWriteOnce` | RWO | Volume can be mounted as read-write by a single node |
| `ReadOnlyMany` | ROX | Volume can be mounted as read-only by many nodes |
| `ReadWriteMany` | RWX | Volume can be mounted as read-write by many nodes |
| `ReadWriteOncePod` | RWOP | Volume can be mounted as read-write by a single Pod |

**Note**: Not all storage providers support all access modes. Check your provider's documentation.

### 2.3 Reclaim Policies

Reclaim policies define what happens to a PV when its PVC is deleted:

| Policy | Description |
|--------|-------------|
| `Retain` | Manual reclamation; PV is not deleted, data preserved |
| `Delete` | PV and underlying storage are deleted |
| `Recycle` | Basic scrub (`rm -rf /thevolume/*`) - **Deprecated** |

#### Changing Reclaim Policy

```bash
# View current reclaim policy
kubectl get pv <pv-name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'

# Patch to change reclaim policy
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

---

## 3. Manage Persistent Volumes (PVs) and Persistent Volume Claims (PVCs)

### 3.1 PersistentVolume (PV)

A PV is a piece of storage in the cluster provisioned by an administrator or dynamically provisioned using StorageClasses.

#### PV Phases

| Phase | Description |
|-------|-------------|
| `Available` | Free resource, not yet bound to a claim |
| `Bound` | Volume is bound to a claim |
| `Released` | Claim deleted, but resource not yet reclaimed |
| `Failed` | Volume failed automatic reclamation |

#### Creating a PV Manually (Static Provisioning)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data"
```

#### NFS PersistentVolume Example

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: nfs-server.example.com
    path: "/exports/data"
  mountOptions:
    - hard
    - nfsvers=4.1
```

### 3.2 PersistentVolumeClaim (PVC)

A PVC is a request for storage by a user. It is similar to a Pod—Pods consume node resources and PVCs consume PV resources.

#### Creating a PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:               # Optional: select specific PV
    matchLabels:
      type: local
```

### 3.3 Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/var/www/html"
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### 3.4 PV to PVC Binding

The binding process:
1. User creates PVC with specific requirements
2. Control loop watches for new PVCs
3. Finds a matching PV (capacity, access modes, storage class)
4. Binds PVC to PV (one-to-one mapping)

**Binding Criteria**:
- Storage capacity (PV must be >= PVC request)
- Access modes (PV must support requested modes)
- StorageClass (must match)
- Volume mode (Filesystem or Block)
- Selector labels (if specified)

### 3.5 Storage Object in Use Protection

Kubernetes protects PVCs in active use and bound PVs from deletion:

```bash
# Check if PVC is protected
kubectl get pvc <pvc-name> -o jsonpath='{.metadata.finalizers}'
# Output: [kubernetes.io/pvc-protection]
```

---

## 4. Essential kubectl Commands for Storage

### Viewing Storage Resources

```bash
# List all StorageClasses
kubectl get storageclass
kubectl get sc

# List PersistentVolumes
kubectl get persistentvolumes
kubectl get pv

# List PersistentVolumeClaims
kubectl get persistentvolumeclaims
kubectl get pvc

# Get detailed information
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>
kubectl describe sc <storageclass-name>

# Get PV bound to a specific PVC
kubectl get pvc <pvc-name> -o jsonpath='{.spec.volumeName}'
```

### Creating Storage Resources Imperatively

```bash
# Create a PVC (use --dry-run to generate YAML)
kubectl create -f pvc.yaml

# Generate PVC YAML template
kubectl create pvc my-pvc --dry-run=client -o yaml > pvc.yaml
```

### Expanding a PVC

```bash
# Edit the PVC and increase spec.resources.requests.storage
kubectl edit pvc <pvc-name>

# Or use patch
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

**Note**: The StorageClass must have `allowVolumeExpansion: true`

---

## 5. Practice Exercises

### Exercise 1: Create a StorageClass

Create a StorageClass named "slow" with the following specifications:
- Provisioner: `kubernetes.io/no-provisioner`
- Reclaim Policy: Retain
- Volume Binding Mode: WaitForFirstConsumer

<details>
<summary>Solution</summary>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

</details>

### Exercise 2: Create PV and PVC

1. Create a PV with 1Gi capacity, ReadWriteOnce access mode, using hostPath `/data/pv1`
2. Create a PVC that requests 500Mi from this PV
3. Create a Pod that uses this PVC

<details>
<summary>Solution</summary>

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-exercise
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/pv1"
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-exercise
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
# Pod
apiVersion: v1
kind: Pod
metadata:
  name: pod-exercise
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-exercise
```

</details>

### Exercise 3: Troubleshoot PVC Pending State

A PVC is stuck in Pending state. What are the possible causes and how do you debug?

<details>
<summary>Solution</summary>

```bash
# Check PVC status and events
kubectl describe pvc <pvc-name>

# Common causes:
# 1. No matching PV available
kubectl get pv

# 2. StorageClass doesn't exist or has no provisioner
kubectl get sc

# 3. Capacity mismatch (PVC requests more than available PVs)
# 4. Access mode mismatch
# 5. Selector labels don't match any PV
# 6. Dynamic provisioning failed - check provisioner logs
```

</details>

---

## 6. Key Points for the Exam

1. **Dynamic vs Static Provisioning**: Know when each is appropriate
2. **StorageClass** is required for dynamic provisioning
3. **Access Modes**: RWO (single node), ROX (read-only many), RWX (read-write many), RWOP (single pod)
4. **Reclaim Policies**: Retain (manual cleanup), Delete (automatic deletion)
5. **PVC Binding**: One-to-one, based on capacity, access modes, and storage class
6. **Volume Expansion**: Requires `allowVolumeExpansion: true` in StorageClass
7. **Protection Finalizers**: Prevent accidental deletion of in-use storage
8. **VolumeBindingMode**: `Immediate` vs `WaitForFirstConsumer` (topology-aware)

---

## 7. Quick Reference Commands

```bash
# Get all storage-related resources
kubectl get sc,pv,pvc --all-namespaces

# Check if a PV is bound
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.claimRef.name

# Check storage class details
kubectl get sc -o yaml

# Force delete a PVC stuck in Terminating
kubectl patch pvc <pvc-name> -p '{"metadata":{"finalizers":null}}'

# Get events related to storage
kubectl get events --field-selector reason=ProvisioningFailed
```

---

## 8. Documentation References

During the exam, you can access:
- https://kubernetes.io/docs/concepts/storage/
- https://kubernetes.io/docs/concepts/storage/storage-classes/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/