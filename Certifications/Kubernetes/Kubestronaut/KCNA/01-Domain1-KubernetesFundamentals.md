# KCNA Exam Study Guide: Domain 1 - Kubernetes Fundamentals

## Domain Weight: 46% (~28 questions out of 60)

This is the most heavily weighted domain in the KCNA exam. You must have a thorough understanding of Kubernetes architecture, core components, objects, and basic kubectl operations.

---

## 1.1 Kubernetes Architecture

### What is Kubernetes?
Kubernetes (K8s) is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. Originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF).

### Cluster Architecture Overview
A Kubernetes cluster consists of two main types of nodes:

1. **Control Plane Nodes (Master)** - Manage the cluster state
2. **Worker Nodes** - Run application workloads

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                            │
│  ┌───────────────┐ ┌───────────────┐ ┌────────────────────────┐ │
│  │ kube-apiserver│ │ kube-scheduler│ │ kube-controller-manager│ │
│  └───────────────┘ └───────────────┘ └────────────────────────┘ │
│  ┌──────────────┐ ┌──────────────────────────────────────────┐  │
│  │     etcd     │ │     cloud-controller-manager (optional)  │  │
│  └──────────────┘ └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        WORKER NODE(S)                           │
│  ┌──────────────┐ ┌──────────────┐ ┌────────────────────────┐   │
│  │    kubelet   │ │  kube-proxy  │ │   Container Runtime    │   │
│  └──────────────┘ └──────────────┘ └────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Pods (Containers)                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Control Plane Components

### kube-apiserver
- **Purpose**: Front-end for the Kubernetes control plane
- **Function**: Exposes the Kubernetes API and handles all API requests
- **Key Points**:
  - All communication goes through the API server
  - Validates and configures data for API objects
  - Only component that directly interacts with etcd
  - Designed to scale horizontally (multiple instances)
  - RESTful interface for CRUD operations

### etcd
- **Purpose**: Consistent and highly-available key-value store
- **Function**: Stores all cluster data (the "source of truth")
- **Key Points**:
  - Stores configuration data, state, and metadata
  - Uses the Raft consensus algorithm for distributed consistency
  - Should be backed up regularly
  - Critical for cluster recovery
  - Typically runs on control plane nodes

### kube-scheduler
- **Purpose**: Assigns Pods to Nodes
- **Function**: Watches for newly created Pods with no assigned node
- **Key Points**:
  - Considers resource requirements (CPU, memory)
  - Evaluates constraints (affinity, anti-affinity, taints, tolerations)
  - Respects node selectors and node affinity rules
  - Does NOT actually run the Pod (kubelet does that)

### kube-controller-manager
- **Purpose**: Runs controller processes
- **Function**: Manages various controllers that regulate cluster state
- **Key Controllers**:
  - **Node Controller**: Monitors node health
  - **Replication Controller**: Maintains correct number of Pod replicas
  - **Endpoints Controller**: Populates Endpoints objects
  - **Service Account & Token Controllers**: Create accounts and API access tokens
  - **Job Controller**: Manages Job objects
  - **Deployment Controller**: Manages Deployments and ReplicaSets

### cloud-controller-manager (Optional)
- **Purpose**: Integrates with cloud provider APIs
- **Function**: Separates cloud-specific control logic from core Kubernetes
- **Key Points**:
  - Only runs in cloud environments
  - Manages cloud-specific controllers (Node, Route, Service)
  - Enables cloud-specific features (load balancers, storage)

---

## 1.3 Worker Node Components

### kubelet
- **Purpose**: Primary node agent that runs on each worker node
- **Function**: Ensures containers are running in Pods
- **Key Points**:
  - Communicates with the API server
  - Receives Pod specifications (PodSpecs) and ensures containers match
  - Reports node and Pod status to the control plane
  - Does NOT manage containers not created by Kubernetes
  - Performs health checks (liveness, readiness probes)

### kube-proxy
- **Purpose**: Network proxy that runs on each node
- **Function**: Maintains network rules for Pod communication
- **Key Points**:
  - Implements the Service abstraction
  - Uses iptables, IPVS, or userspace mode for traffic routing
  - Handles TCP, UDP, and SCTP traffic
  - Enables Service discovery and load balancing

### Container Runtime
- **Purpose**: Software responsible for running containers
- **Function**: Pulls images and runs containers
- **Key Points**:
  - Must implement the Container Runtime Interface (CRI)
  - Examples: containerd, CRI-O, Docker (via cri-dockerd)
  - Manages container lifecycle (create, start, stop, delete)

---

## 1.4 Kubernetes Objects (Resources)

### Understanding Kubernetes Objects
- **Definition**: Persistent entities in the Kubernetes system representing the cluster's state
- **Purpose**: Describe what containerized applications are running, their resources, and policies
- Objects have two main fields:
  - **spec**: Desired state (what you want)
  - **status**: Current state (what exists)

### Pods
**The smallest deployable unit in Kubernetes**

- Contains one or more containers with shared storage and network
- Containers in a Pod share:
  - Network namespace (same IP address)
  - Storage volumes
  - Lifecycle (scheduled together)
- Ephemeral by nature (not persistent)
- Types:
  - **Single-container Pod**: Most common use case
  - **Multi-container Pod**: Tightly coupled containers (sidecar pattern)

**Pod YAML Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
spec:
  containers:
  - name: my-container
    image: nginx:1.21
    ports:
    - containerPort: 80
```

### ReplicaSets
- **Purpose**: Maintain a stable set of replica Pods
- **Function**: Ensures a specified number of identical Pods are running
- **Key Points**:
  - Uses label selectors to identify Pods
  - Automatically creates or deletes Pods to match desired count
  - Usually managed by Deployments (not created directly)

### Deployments
**Recommended way to manage Pods and ReplicaSets**

- **Purpose**: Declarative updates for Pods and ReplicaSets
- **Features**:
  - Rolling updates with zero downtime
  - Rollback capabilities
  - Scaling (up/down)
  - Pause and resume updates

**Deployment YAML Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 80
```

**Deployment Strategies:**
- **RollingUpdate** (default): Gradually replaces old Pods with new ones
- **Recreate**: Kills all existing Pods before creating new ones

### Services
**Stable network endpoint for accessing Pods**

- **Purpose**: Abstract way to expose applications running on Pods
- **Key Points**:
  - Provides stable IP address and DNS name
  - Load balances traffic across Pod replicas
  - Decouples frontend from backend

**Service Types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **ClusterIP** | Internal cluster IP only (default) | Internal services |
| **NodePort** | Exposes on each node's IP at a static port | Development/testing |
| **LoadBalancer** | Cloud provider load balancer | Production (cloud) |
| **ExternalName** | Maps to external DNS name | External services |

**Service YAML Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

### ConfigMaps
- **Purpose**: Store non-confidential configuration data
- **Function**: Decouple configuration from container images
- **Usage**: Environment variables, command-line arguments, or config files

**ConfigMap Example:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "mysql://db:3306"
  log_level: "INFO"
```

### Secrets
- **Purpose**: Store sensitive data (passwords, tokens, keys)
- **Key Points**:
  - Base64 encoded (NOT encrypted by default)
  - Should be encrypted at rest in etcd
  - Mounted as files or environment variables

**Types of Secrets:**
- `Opaque`: User-defined data (default)
- `kubernetes.io/service-account-token`: Service account tokens
- `kubernetes.io/dockerconfigjson`: Docker registry credentials
- `kubernetes.io/tls`: TLS certificates

### Namespaces
- **Purpose**: Virtual clusters within a physical cluster
- **Function**: Divide cluster resources between multiple users/teams
- **Default Namespaces**:
  - `default`: Default namespace for objects with no namespace
  - `kube-system`: Kubernetes system components
  - `kube-public`: Publicly accessible resources
  - `kube-node-lease`: Node heartbeat leases

### DaemonSets
- **Purpose**: Ensure a Pod runs on all (or selected) nodes
- **Use Cases**:
  - Log collectors (Fluentd, Filebeat)
  - Monitoring agents (Prometheus Node Exporter)
  - Storage daemons
  - Network plugins

### StatefulSets
- **Purpose**: Manage stateful applications
- **Features**:
  - Stable, unique network identifiers
  - Stable, persistent storage
  - Ordered, graceful deployment and scaling
- **Use Cases**: Databases, ZooKeeper, Kafka

### Jobs and CronJobs
**Jobs:**
- Run Pods to completion (batch processing)
- Ensures specified number of successful completions

**CronJobs:**
- Schedule Jobs to run periodically
- Uses cron-like schedule syntax

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
          restartPolicy: OnFailure
```

---

## 1.5 Labels and Selectors

### Labels
- **Definition**: Key-value pairs attached to objects
- **Purpose**: Organize and select subsets of objects
- **Characteristics**:
  - Can be added at creation or later
  - Multiple labels per object
  - Used by selectors to identify objects

**Example Labels:**
```yaml
metadata:
  labels:
    app: myapp
    environment: production
    tier: frontend
    version: v1.2.3
```

### Selectors
- **Purpose**: Query and filter objects based on labels
- **Types**:
  - **Equality-based**: `=`, `==`, `!=`
  - **Set-based**: `in`, `notin`, `exists`

**Examples:**
```bash
# Equality-based
kubectl get pods -l environment=production

# Set-based
kubectl get pods -l 'environment in (production, staging)'
```

---

## 1.6 The Kubernetes API

### API Overview
- RESTful interface for all Kubernetes operations
- Organized into API groups (core, apps, batch, etc.)
- Versioned (v1, v1beta1, v1alpha1)

### API Groups
| Group | Resources |
|-------|-----------|
| Core (v1) | Pods, Services, ConfigMaps, Secrets, Namespaces |
| apps/v1 | Deployments, ReplicaSets, StatefulSets, DaemonSets |
| batch/v1 | Jobs, CronJobs |
| networking.k8s.io/v1 | NetworkPolicies, Ingress |
| rbac.authorization.k8s.io/v1 | Roles, RoleBindings, ClusterRoles |

### API Request Flow
1. Authentication (Who are you?)
2. Authorization (What can you do?)
3. Admission Control (Should this be allowed?)
4. Validation and Persistence

---

## 1.7 kubectl - The Kubernetes CLI

### Essential kubectl Commands

**Cluster Information:**
```bash
kubectl cluster-info                 # Display cluster info
kubectl get nodes                    # List all nodes
kubectl describe node <node-name>    # Detailed node info
kubectl api-resources                # List all API resources
kubectl api-versions                 # List API versions
```

**Working with Pods:**
```bash
kubectl get pods                     # List pods in current namespace
kubectl get pods -A                  # List pods in all namespaces
kubectl get pods -o wide             # List with additional info
kubectl describe pod <pod-name>      # Detailed pod info
kubectl logs <pod-name>              # View pod logs
kubectl logs -f <pod-name>           # Stream logs
kubectl exec -it <pod-name> -- /bin/sh  # Execute shell in pod
kubectl delete pod <pod-name>        # Delete a pod
```

**Working with Deployments:**
```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.21
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
```

**Working with Services:**
```bash
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get services
kubectl describe service nginx
```

**Declarative Management:**
```bash
kubectl apply -f manifest.yaml       # Create/update resources
kubectl delete -f manifest.yaml      # Delete resources
kubectl diff -f manifest.yaml        # Show changes
```

**Configuration and Context:**
```bash
kubectl config view                  # View kubeconfig
kubectl config get-contexts          # List contexts
kubectl config use-context <name>    # Switch context
kubectl config set-context --current --namespace=<ns>
```

---

## 1.8 Scheduling Concepts

### How Scheduling Works
1. User creates a Pod (or Deployment creates it)
2. Pod enters the scheduling queue (Pending state)
3. Scheduler evaluates:
   - **Filtering**: Which nodes can run the Pod?
   - **Scoring**: Which node is best?
4. Pod is bound to selected node
5. kubelet runs the Pod on that node

### Scheduling Factors
- **Resource Requests and Limits**
- **Node Selectors**: Simple node selection by labels
- **Node Affinity**: More expressive node selection rules
- **Pod Affinity/Anti-Affinity**: Schedule based on other Pods
- **Taints and Tolerations**: Repel Pods from nodes
- **Pod Priority and Preemption**

### Node Selector Example
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### Taints and Tolerations
**Taint a Node:**
```bash
kubectl taint nodes node1 key=value:NoSchedule
```

**Toleration in Pod:**
```yaml
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

---

## 1.9 Container Basics

### What is a Container?
- Lightweight, standalone executable package
- Includes application code, runtime, libraries, and settings
- Shares the host OS kernel (unlike VMs)
- Provides process isolation using Linux namespaces and cgroups

### Container vs Virtual Machine

| Aspect | Container | Virtual Machine |
|--------|-----------|-----------------|
| Isolation | Process-level | Hardware-level |
| OS | Shares host kernel | Full OS per VM |
| Size | Megabytes | Gigabytes |
| Startup | Seconds | Minutes |
| Overhead | Low | High |

### Container Images
- **Definition**: Read-only template used to create containers
- **Layers**: Images are built in layers (efficient storage/transfer)
- **Registry**: Storage location for images (Docker Hub, ECR, GCR, Quay.io)

### Image Naming Convention
```
registry/repository:tag
```
Examples:
- `nginx:1.21`
- `docker.io/library/nginx:latest`
- `gcr.io/project/myapp:v1.2.3`

### Container Lifecycle in Kubernetes
1. **Image Pull**: kubelet pulls the container image
2. **Container Creation**: Container runtime creates the container
3. **Running**: Container executes its main process
4. **Termination**: Container stops (success or failure)

---

## 1.10 Probes and Health Checks

### Types of Probes

**Liveness Probe:**
- Determines if container is running
- If fails, kubelet kills and restarts the container
- Use for deadlock detection

**Readiness Probe:**
- Determines if container is ready to receive traffic
- If fails, Pod is removed from Service endpoints
- Use for startup delays or dependency checks

**Startup Probe:**
- Determines if application has started
- Disables liveness/readiness until successful
- Use for slow-starting applications

### Probe Methods
1. **HTTP GET**: Probe makes HTTP request to container
2. **TCP Socket**: Probe opens TCP connection
3. **Exec**: Probe executes command in container
4. **gRPC**: Probe performs gRPC health check

### Probe Configuration Example
```yaml
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

---

## Key Exam Tips for This Domain

1. **Know the Control Plane components** and their specific functions
2. **Understand the difference** between ReplicaSet, Deployment, and StatefulSet
3. **Master basic kubectl commands** - they appear frequently
4. **Understand Service types** and when to use each
5. **Know the difference** between ConfigMaps and Secrets
6. **Understand Pod lifecycle** and container states
7. **Know what Labels and Selectors** are used for
8. **Understand scheduling concepts** (taints, tolerations, node selectors)
9. **Know the probe types** and their purposes

---

## Practice Questions

1. Which component is responsible for scheduling Pods to nodes?
   - Answer: kube-scheduler

2. What is the default Service type in Kubernetes?
   - Answer: ClusterIP

3. Which component stores all cluster data?
   - Answer: etcd

4. What is the smallest deployable unit in Kubernetes?
   - Answer: Pod

5. Which probe determines if a container is ready to receive traffic?
   - Answer: Readiness Probe

6. What command lists all Pods in all namespaces?
   - Answer: `kubectl get pods -A` or `kubectl get pods --all-namespaces`

7. Which resource ensures a Pod runs on every node?
   - Answer: DaemonSet

8. What does the kubelet do?
   - Answer: Runs on each node and ensures containers are running in Pods

---

## Additional Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)