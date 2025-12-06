# CKA Study Guide: Workloads and Scheduling (15%)

## Domain Overview

The Workloads and Scheduling domain covers deploying and managing applications in Kubernetes, including Deployments, rolling updates, rollbacks, ConfigMaps, Secrets, autoscaling, self-healing mechanisms, and pod admission/scheduling controls.

---

## 1. Understand Application Deployments and Rolling Updates/Rollbacks

### 1.1 Deployments

A **Deployment** provides declarative updates for Pods and ReplicaSets. You describe a desired state, and the Deployment controller changes the actual state to the desired state.

#### Creating a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

#### Imperative Deployment Commands

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# Generate YAML without creating
kubectl create deployment nginx --image=nginx:1.21 --dry-run=client -o yaml > deployment.yaml

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Set image
kubectl set image deployment/nginx nginx=nginx:1.22

# View deployment status
kubectl rollout status deployment/nginx
```

### 1.2 Rolling Updates

Rolling updates allow Deployments to update with zero downtime by incrementally updating Pod instances with new ones.

#### Rolling Update Strategy Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Max pods above desired count during update
      maxUnavailable: 25%  # Max pods unavailable during update
  # ... rest of spec
```

#### Strategy Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `maxSurge` | Max additional pods during update (absolute or %) | 25% |
| `maxUnavailable` | Max unavailable pods during update (absolute or %) | 25% |
| `type` | RollingUpdate or Recreate | RollingUpdate |

#### Performing a Rolling Update

```bash
# Update image (triggers rolling update)
kubectl set image deployment/nginx nginx=nginx:1.22

# Update using edit
kubectl edit deployment nginx

# Update using patch
kubectl patch deployment nginx -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.23"}]}}}}'

# Watch the rollout progress
kubectl rollout status deployment/nginx

# View rollout history
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision=2
```

### 1.3 Rollbacks

```bash
# Rollback to previous revision
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# Pause a rollout
kubectl rollout pause deployment/nginx

# Resume a rollout
kubectl rollout resume deployment/nginx

# View revision history
kubectl rollout history deployment/nginx

# Annotate deployment change cause (for better history)
kubectl annotate deployment/nginx kubernetes.io/change-cause="Updated to nginx 1.22"
```

### 1.4 Recreate Strategy

The Recreate strategy kills all existing Pods before creating new ones (causes downtime).

```yaml
spec:
  strategy:
    type: Recreate
```

---

## 2. Use ConfigMaps and Secrets to Configure Applications

### 2.1 ConfigMaps

ConfigMaps store non-confidential configuration data as key-value pairs.

#### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info

# From file
kubectl create configmap app-config --from-file=config.properties

# From directory
kubectl create configmap app-config --from-file=config/

# From env file
kubectl create configmap app-config --from-env-file=app.env

# Generate YAML
kubectl create configmap app-config --from-literal=key=value --dry-run=client -o yaml
```

#### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.json: |
    {
      "database": "mydb",
      "port": 5432
    }
```

#### Using ConfigMaps in Pods

**As Environment Variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    # Single key
    env:
    - name: APP_ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    # All keys from ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
```

**As Volume Mount:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      # Optional: specific items
      items:
      - key: config.json
        path: app-config.json
```

### 2.2 Secrets

Secrets store sensitive data like passwords, tokens, and keys.

#### Creating Secrets

```bash
# Generic secret from literals
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret123

# From file
kubectl create secret generic tls-secret --from-file=tls.crt --from-file=tls.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=user \
  --docker-password=password \
  --docker-email=email@example.com

# TLS secret
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

#### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Default; arbitrary user-defined data |
| `kubernetes.io/service-account-token` | ServiceAccount token |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/basic-auth` | Basic authentication credentials |
| `kubernetes.io/ssh-auth` | SSH private key |

#### Secret YAML (base64 encoded)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=      # base64 encoded "admin"
  password: c2VjcmV0MTIz  # base64 encoded "secret123"
---
# Alternative: stringData (plain text, encoded automatically)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: secret123
```

#### Using Secrets in Pods

**As Environment Variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

**As Volume Mount:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400  # File permissions
```

#### Base64 Encoding/Decoding

```bash
# Encode
echo -n 'admin' | base64
# Output: YWRtaW4=

# Decode
echo 'YWRtaW4=' | base64 --decode
# Output: admin

# Get secret and decode
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode
```

---

## 3. Configure Workload Autoscaling

### 3.1 Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of Pods based on observed metrics (CPU, memory, or custom metrics).

#### Creating HPA

```bash
# Create HPA imperatively
kubectl autoscale deployment nginx --cpu-percent=50 --min=1 --max=10

# View HPA status
kubectl get hpa
kubectl describe hpa nginx
```

#### HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

#### HPA Requirements

- **Metrics Server must be installed** for resource metrics
- Pods must have **resource requests** defined
- Custom metrics require a metrics adapter

### 3.2 Vertical Pod Autoscaler (VPA)

VPA automatically adjusts CPU and memory requests/limits for Pods.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: "100m"
        memory: "50Mi"
      maxAllowed:
        cpu: "1"
        memory: "500Mi"
```

---

## 4. Understand Primitives for Self-Healing Application Deployments

### 4.1 Self-Healing Mechanisms

Kubernetes provides automatic self-healing capabilities:

| Mechanism | Description |
|-----------|-------------|
| **Container restarts** | Kubelet restarts failed containers based on `restartPolicy` |
| **Pod replacement** | Controllers (Deployment, ReplicaSet, DaemonSet) replace failed Pods |
| **Node failure handling** | Pods are rescheduled to healthy nodes |
| **Health checks** | Liveness/readiness probes detect unhealthy containers |

### 4.2 Restart Policies

```yaml
spec:
  restartPolicy: Always    # Default for Deployments
  # Options: Always, OnFailure, Never
```

| Policy | Use Case |
|--------|----------|
| `Always` | Long-running applications (default for Deployments) |
| `OnFailure` | Jobs that should retry on failure |
| `Never` | Jobs that should not retry |

### 4.3 Liveness Probes

Liveness probes detect when a container needs to be restarted.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
      successThreshold: 1
```

#### Probe Types

```yaml
# HTTP GET probe
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Value

# TCP Socket probe
livenessProbe:
  tcpSocket:
    port: 3306

# Exec probe
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy

# gRPC probe
livenessProbe:
  grpc:
    port: 50051
```

### 4.4 Readiness Probes

Readiness probes determine when a container is ready to accept traffic.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 4.5 Startup Probes

Startup probes for slow-starting containers.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
  # Container has 30 * 10 = 300 seconds to start
```

---

## 5. Configure Pod Admission and Scheduling

### 5.1 Resource Requests and Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: myapp
    resources:
      requests:        # Minimum guaranteed resources
        memory: "64Mi"
        cpu: "250m"
      limits:          # Maximum allowed resources
        memory: "128Mi"
        cpu: "500m"
```

#### Resource Units

| Resource | Units |
|----------|-------|
| CPU | `m` (millicores), `1` = 1000m |
| Memory | `Ki`, `Mi`, `Gi`, `Ti` (binary); `K`, `M`, `G`, `T` (decimal) |

### 5.2 Node Selectors

Simple node selection based on labels.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
```

```bash
# Label a node
kubectl label nodes <node-name> disktype=ssd
```

### 5.3 Node Affinity

More expressive node selection rules.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

#### Affinity Operators

| Operator | Description |
|----------|-------------|
| `In` | Label value in set |
| `NotIn` | Label value not in set |
| `Exists` | Label exists |
| `DoesNotExist` | Label doesn't exist |
| `Gt` | Greater than (numeric) |
| `Lt` | Less than (numeric) |

### 5.4 Pod Affinity and Anti-Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: web
    image: nginx
```

### 5.5 Taints and Tolerations

**Taints** allow nodes to repel certain pods.
**Tolerations** allow pods to schedule on tainted nodes.

```bash
# Add taint to node
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 dedicated=gpu:NoExecute

# Remove taint
kubectl taint nodes node1 key=value:NoSchedule-
```

#### Taint Effects

| Effect | Description |
|--------|-------------|
| `NoSchedule` | Pods won't be scheduled unless they tolerate the taint |
| `PreferNoSchedule` | Scheduler tries to avoid but doesn't guarantee |
| `NoExecute` | Evicts existing pods that don't tolerate the taint |

#### Tolerations in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  - key: "key2"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 3600  # Evict after 1 hour
  containers:
  - name: app
    image: myapp
```

### 5.6 Pod Topology Spread Constraints

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spread-pod
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  containers:
  - name: nginx
    image: nginx
```

---

## 6. Other Workload Resources

### 6.1 ReplicaSet

Maintains a stable set of replica Pods. Usually managed by Deployments.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

### 6.2 DaemonSet

Ensures all (or some) nodes run a copy of a Pod.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd
```

### 6.3 StatefulSet

For stateful applications requiring stable network identities and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
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
        image: mysql:5.7
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 6.4 Jobs and CronJobs

```yaml
# Job
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 4
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
---
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool
          restartPolicy: OnFailure
```

---

## 7. Practice Exercises

### Exercise 1: Create and Scale a Deployment

Create a deployment named `webapp` with image `nginx:1.20`, 3 replicas, then update to `nginx:1.21` and perform a rollback.

<details>
<summary>Solution</summary>

```bash
# Create deployment
kubectl create deployment webapp --image=nginx:1.20 --replicas=3

# Update image
kubectl set image deployment/webapp nginx=nginx:1.21

# Check rollout status
kubectl rollout status deployment/webapp

# View history
kubectl rollout history deployment/webapp

# Rollback
kubectl rollout undo deployment/webapp
```

</details>

### Exercise 2: ConfigMap and Secret

Create a pod that uses a ConfigMap for `APP_MODE=production` and a Secret for `DB_PASS=secret123`.

<details>
<summary>Solution</summary>

```bash
# Create ConfigMap
kubectl create configmap app-config --from-literal=APP_MODE=production

# Create Secret
kubectl create secret generic db-secret --from-literal=DB_PASS=secret123
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_MODE
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASS
```

</details>

### Exercise 3: Schedule Pod on Specific Node

Create a pod that only schedules on nodes with label `gpu=true`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - name: app
    image: nvidia/cuda
```

Or with node affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
  containers:
  - name: app
    image: nvidia/cuda
```

</details>

---

## 8. Key Points for the Exam

1. **Deployments**: Know how to create, scale, update, and rollback
2. **Rolling updates**: Understand `maxSurge` and `maxUnavailable`
3. **ConfigMaps**: Environment variables and volume mounts
4. **Secrets**: Base64 encoding, types, and usage
5. **HPA**: Requires metrics-server and resource requests
6. **Probes**: Liveness, Readiness, and Startup probes
7. **Scheduling**: nodeSelector, affinity, taints/tolerations
8. **Resource management**: requests vs limits

---

## 9. Documentation References

- https://kubernetes.io/docs/concepts/workloads/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/concepts/scheduling-eviction/