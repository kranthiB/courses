# KCNA Exam Study Guide: Domain 2 - Container Orchestration

## Domain Weight: 22% (~13 questions out of 60)

This domain covers the fundamentals of container orchestration, including container runtimes, networking, security, storage, and service mesh concepts.

---

## 2.1 Container Orchestration Fundamentals

### What is Container Orchestration?
Container orchestration is the automated management of containerized applications across multiple hosts. It handles:

- **Deployment**: Scheduling containers across a cluster
- **Scaling**: Adding/removing container instances based on demand
- **Networking**: Enabling container communication
- **Load Balancing**: Distributing traffic across containers
- **Health Monitoring**: Detecting and replacing failed containers
- **Rolling Updates**: Updating applications with zero downtime

### Why Kubernetes for Container Orchestration?
- **Self-healing**: Automatically restarts failed containers
- **Horizontal scaling**: Scale based on CPU, memory, or custom metrics
- **Service discovery and load balancing**: Built-in DNS and networking
- **Automated rollouts and rollbacks**: Declarative update management
- **Secret and configuration management**: Secure storage for sensitive data
- **Storage orchestration**: Automatic mounting of storage systems

### Container Orchestration Platforms
| Platform | Description |
|----------|-------------|
| **Kubernetes** | Industry standard, CNCF graduated project |
| **Docker Swarm** | Docker's native orchestration tool |
| **Apache Mesos** | Distributed systems kernel with Marathon for containers |
| **Nomad** | HashiCorp's workload orchestrator |
| **Amazon ECS** | AWS-managed container orchestration |

---

## 2.2 Container Runtime

### Container Runtime Interface (CRI)
- **Definition**: Standard API between Kubernetes and container runtimes
- **Purpose**: Allows Kubernetes to support multiple container runtimes
- **Components**:
  - **RuntimeService**: Manages Pod and container lifecycle
  - **ImageService**: Manages container images

### Common Container Runtimes

#### containerd
- **Type**: High-level runtime (implements CRI)
- **Origin**: Extracted from Docker, now a CNCF graduated project
- **Features**:
  - Manages complete container lifecycle
  - Image pull and push operations
  - Storage and network attachments
  - Default runtime for many Kubernetes distributions

#### CRI-O
- **Type**: Lightweight runtime specifically for Kubernetes
- **Origin**: Red Hat, CNCF incubating project
- **Features**:
  - Minimal and optimized for Kubernetes
  - OCI-compliant
  - Used by OpenShift and other distributions

#### Docker Engine (via cri-dockerd)
- **Note**: Kubernetes removed direct Docker support in v1.24
- **Solution**: Use cri-dockerd adapter to continue using Docker
- Docker uses containerd under the hood anyway

### Low-Level vs High-Level Runtimes

```
┌────────────────────────────────────────────┐
│             Kubernetes (kubelet)           │
└────────────────────────────────────────────┘
                    │ CRI
                    ▼
┌────────────────────────────────────────────┐
│    High-Level Runtime (containerd/CRI-O)   │
└────────────────────────────────────────────┘
                    │ OCI
                    ▼
┌────────────────────────────────────────────┐
│     Low-Level Runtime (runc/crun/gVisor)   │
└────────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────┐
│          Linux Kernel (cgroups, namespaces)│
└────────────────────────────────────────────┘
```

### OCI (Open Container Initiative)
- **Purpose**: Create open standards for containers
- **Specifications**:
  - **Runtime Spec**: How to run containers
  - **Image Spec**: How to build and package container images
  - **Distribution Spec**: How to distribute images

---

## 2.3 Container Networking

### Kubernetes Networking Model
The Kubernetes networking model requires:
1. **Pod-to-Pod**: All Pods can communicate without NAT
2. **Node-to-Pod**: All nodes can communicate with all Pods without NAT
3. **Pod IP Consistency**: A Pod sees the same IP that others see for it

### Container Network Interface (CNI)
- **Definition**: Standard for configuring network interfaces in Linux containers
- **Purpose**: Provides a plugin-based networking solution for Kubernetes
- **How it works**: When a Pod is created, kubelet calls the CNI plugin to set up networking

### Popular CNI Plugins

| Plugin | Description | Key Features |
|--------|-------------|--------------|
| **Calico** | Layer 3 networking with network policy | BGP routing, network policies, high performance |
| **Flannel** | Simple overlay network | Easy setup, vxlan/host-gw backends |
| **Weave Net** | Mesh networking | Encryption, multicast support |
| **Cilium** | eBPF-based networking | Advanced security, observability, high performance |
| **AWS VPC CNI** | Native AWS networking | Uses AWS ENI for Pod IPs |

### Network Policies
- **Purpose**: Control traffic flow at the Pod level (L3/L4)
- **Default**: All traffic is allowed between Pods
- **Key Concepts**:
  - **Ingress**: Incoming traffic to a Pod
  - **Egress**: Outgoing traffic from a Pod
  - **Selectors**: Label-based Pod selection

**Network Policy Example:**
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

### CoreDNS
- **Purpose**: Kubernetes DNS server for service discovery
- **Function**: Resolves Service names to ClusterIP addresses
- **DNS Records**:
  - Services: `<service>.<namespace>.svc.cluster.local`
  - Pods: `<pod-ip>.<namespace>.pod.cluster.local`

**Example DNS Resolution:**
```
my-service.default.svc.cluster.local → 10.96.0.1
```

### Kubernetes Service Networking

**ClusterIP (default):**
- Internal cluster IP only
- Not accessible from outside cluster

**NodePort:**
- Opens a port (30000-32767) on all nodes
- External access: `<NodeIP>:<NodePort>`

**LoadBalancer:**
- Provisions cloud provider load balancer
- External access through load balancer IP

**Headless Service (ClusterIP: None):**
- No single cluster IP
- Returns Pod IPs directly
- Used for StatefulSets and service discovery

---

## 2.4 Container Security

### Security Best Practices

#### Pod Security Standards (PSS)
Three security levels defined by Kubernetes:

| Level | Description | Use Case |
|-------|-------------|----------|
| **Privileged** | No restrictions | System-level workloads |
| **Baseline** | Minimal restrictions, prevents known privilege escalations | Standard workloads |
| **Restricted** | Heavily restricted, security hardened | Security-critical workloads |

#### Pod Security Admission (PSA)
- **Purpose**: Enforce Pod Security Standards at namespace level
- **Modes**:
  - `enforce`: Reject non-compliant Pods
  - `audit`: Log violations
  - `warn`: Display warnings

**Namespace Configuration Example:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### Role-Based Access Control (RBAC)

#### RBAC Components

**Role** (namespace-scoped):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**ClusterRole** (cluster-scoped):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

**RoleBinding** (binds Role to subjects):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding** (cluster-wide binding):
- Binds ClusterRole to subjects across all namespaces

#### RBAC Verbs
| Verb | Description |
|------|-------------|
| `get` | Read a specific resource |
| `list` | List resources |
| `watch` | Watch for changes |
| `create` | Create new resources |
| `update` | Modify existing resources |
| `patch` | Partially modify resources |
| `delete` | Delete resources |
| `deletecollection` | Delete a collection |

### Security Context

**Pod-Level Security Context:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

**Key Security Context Settings:**
- `runAsUser`: UID to run the container
- `runAsNonRoot`: Ensure container doesn't run as root
- `readOnlyRootFilesystem`: Make root filesystem read-only
- `allowPrivilegeEscalation`: Prevent privilege escalation
- `capabilities`: Add or drop Linux capabilities

### Secrets Management
- **Best Practices**:
  - Enable encryption at rest for etcd
  - Use external secret management (HashiCorp Vault, AWS Secrets Manager)
  - Limit Secret access with RBAC
  - Avoid storing Secrets in source control
  - Use short-lived credentials when possible

---

## 2.5 Service Mesh

### What is a Service Mesh?
A dedicated infrastructure layer that handles service-to-service communication, providing:
- **Traffic Management**: Load balancing, routing, retries
- **Security**: Mutual TLS (mTLS), authentication, authorization
- **Observability**: Metrics, tracing, logging

### Service Mesh Architecture

```
┌────────────────────────────────────────────────────────────┐
│                      Control Plane                         │
│  (Configuration, Policy, Certificate Management)           │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Service A  │    │  Service B  │    │  Service C  │
│ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │
│ │  App    │ │    │ │  App    │ │    │ │  App    │ │
│ ├─────────┤ │    │ ├─────────┤ │    │ ├─────────┤ │
│ │ Sidecar │◄├────┼►│ Sidecar │◄├────┼►│ Sidecar │ │
│ │ Proxy   │ │    │ │ Proxy   │ │    │ │ Proxy   │ │
│ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │
└─────────────┘    └─────────────┘    └─────────────┘
                     Data Plane
```

### Popular Service Meshes

#### Istio
- **Status**: CNCF Graduated
- **Components**:
  - **istiod**: Control plane (Pilot, Citadel, Galley combined)
  - **Envoy**: Sidecar proxy (data plane)
- **Features**:
  - Traffic management (routing, load balancing)
  - Security (mTLS, authorization policies)
  - Observability (metrics, tracing, logs)
  - Multi-cluster support

#### Linkerd
- **Status**: CNCF Graduated
- **Key Features**:
  - Lightweight and simple
  - Written in Rust (ultra-fast proxy)
  - Automatic mTLS
  - Built-in observability dashboard
  - Low resource overhead

#### Other Service Meshes
| Service Mesh | Description |
|--------------|-------------|
| **Consul Connect** | HashiCorp's service mesh |
| **Cilium** | eBPF-based, can work without sidecars |
| **AWS App Mesh** | AWS-managed service mesh |
| **Kuma** | Universal service mesh (Kubernetes + VMs) |

### Service Mesh Capabilities

**Traffic Management:**
- Canary deployments
- A/B testing
- Traffic mirroring
- Circuit breaking
- Retries and timeouts

**Security:**
- Automatic mTLS encryption
- Fine-grained access control
- Certificate management
- Identity verification

**Observability:**
- Request metrics (latency, throughput, error rates)
- Distributed tracing
- Service dependency visualization

---

## 2.6 Persistent Storage

### Storage Concepts

#### Volumes
- **Definition**: Directory accessible to containers in a Pod
- **Lifetime**: Same as the Pod
- **Types**: emptyDir, hostPath, configMap, secret, persistentVolumeClaim

**EmptyDir Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-emptydir
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: shared-data
  volumes:
  - name: shared-data
    emptyDir: {}
```

#### Persistent Volumes (PV)
- **Definition**: Cluster-level storage resource
- **Lifecycle**: Independent of any Pod
- **Provisioning**:
  - **Static**: Administrator pre-creates PVs
  - **Dynamic**: Automatically created via StorageClass

**Persistent Volume Example:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/pv
```

#### Persistent Volume Claims (PVC)
- **Definition**: Request for storage by a user
- **Purpose**: Binds to a PV that satisfies the request

**PVC Example:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### Access Modes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| **ReadWriteOnce** | RWO | Mounted by single node for read-write |
| **ReadOnlyMany** | ROX | Mounted by multiple nodes for read-only |
| **ReadWriteMany** | RWX | Mounted by multiple nodes for read-write |
| **ReadWriteOncePod** | RWOP | Mounted by single Pod for read-write |

### Reclaim Policies

| Policy | Description |
|--------|-------------|
| **Retain** | Keep PV and data after PVC deletion |
| **Delete** | Delete PV and underlying storage |
| **Recycle** | Basic scrub (deprecated) |

### Container Storage Interface (CSI)
- **Purpose**: Standard for exposing storage systems to Kubernetes
- **Benefits**:
  - Storage vendor independence
  - Dynamic provisioning
  - Snapshots and cloning
  - Volume expansion

**Popular CSI Drivers:**
- AWS EBS CSI Driver
- Google Persistent Disk CSI
- Azure Disk CSI
- Ceph CSI
- NetApp Trident

### Storage Classes
- **Purpose**: Define classes of storage with different capabilities
- **Dynamic Provisioning**: Automatically create PVs

**StorageClass Example:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

## 2.7 Workload Scaling

### Horizontal Pod Autoscaler (HPA)
- **Purpose**: Automatically scale number of Pods
- **Metrics**: CPU, memory, or custom metrics

**HPA Example:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Vertical Pod Autoscaler (VPA)
- **Purpose**: Automatically adjust Pod resource requests/limits
- **Components**:
  - Recommender: Calculates recommended resources
  - Updater: Evicts Pods that need resource updates
  - Admission Controller: Sets resources on new Pods

### Cluster Autoscaler
- **Purpose**: Automatically adjust number of nodes
- **Actions**:
  - **Scale Up**: Add nodes when Pods are pending due to insufficient resources
  - **Scale Down**: Remove underutilized nodes

---

## Key Exam Tips for This Domain

1. **Know the container runtime interfaces** (CRI, OCI) and common runtimes
2. **Understand CNI plugins** and their basic purpose
3. **Master Network Policy** concepts (ingress/egress)
4. **Know RBAC components**: Roles, ClusterRoles, Bindings
5. **Understand Service Mesh** basics and why they're used
6. **Know PV/PVC lifecycle** and access modes
7. **Understand the differences** between service meshes (Istio, Linkerd)
8. **Know security contexts** and Pod Security Standards

---

## Practice Questions

1. What is the purpose of the Container Runtime Interface (CRI)?
   - Answer: It's a standard API that allows Kubernetes to work with multiple container runtimes

2. Which CNI plugin uses eBPF technology for networking?
   - Answer: Cilium

3. What is the default access mode for a PersistentVolume?
   - Answer: There is no default; it must be specified

4. What does a Service Mesh sidecar proxy do?
   - Answer: Intercepts and manages all network traffic to/from the application container

5. Which RBAC resource is namespace-scoped?
   - Answer: Role (and RoleBinding)

6. What is mTLS in the context of service mesh?
   - Answer: Mutual TLS - both client and server verify each other's certificates

7. Name two CNCF graduated service mesh projects.
   - Answer: Istio and Linkerd

8. What component handles DNS in Kubernetes?
   - Answer: CoreDNS

---

## Additional Resources

- [Kubernetes Networking Documentation](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [Linkerd Documentation](https://linkerd.io/docs/)