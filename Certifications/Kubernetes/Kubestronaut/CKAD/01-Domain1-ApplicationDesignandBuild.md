# CKAD Domain 1: Application Design and Build (20%)

## Overview

This domain focuses on your ability to design and build cloud-native applications for Kubernetes. You must be comfortable working with container images, selecting appropriate workload resources, designing multi-container Pods, and managing storage with both persistent and ephemeral volumes.

---

## 1. Define, Build, and Modify Container Images

### Understanding Container Images

Container images are the foundation of Kubernetes workloads. They are immutable, layered filesystems that contain everything needed to run an application: code, runtime, libraries, and dependencies.

### Dockerfile Basics

A Dockerfile defines how to build a container image. Key instructions include:

```dockerfile
# Base image - always start with FROM
FROM nginx:alpine

# Set working directory
WORKDIR /app

# Copy files from build context to image
COPY index.html /usr/share/nginx/html/index.html
COPY src/ /app/src/

# Run commands during build
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV APP_ENV=production
ENV PORT=8080

# Expose ports (documentation only)
EXPOSE 8080

# Define the default command
CMD ["nginx", "-g", "daemon off;"]

# Or use ENTRYPOINT for fixed commands
ENTRYPOINT ["python", "app.py"]
```

### Building Images

```bash
# Build an image with a tag
docker build -t myapp:v1 .

# Build with a specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp:v1 .

# Build for multiple platforms
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:v1 .
```

### Pushing Images to a Registry

```bash
# Tag image for a registry
docker tag myapp:v1 docker.io/username/myapp:v1

# Push to Docker Hub
docker push docker.io/username/myapp:v1

# Push to a private registry
docker tag myapp:v1 registry.example.com/myapp:v1
docker push registry.example.com/myapp:v1
```

### Best Practices for Container Images

1. **Use minimal base images**: Alpine-based images reduce attack surface and size
2. **Multi-stage builds**: Separate build and runtime environments
3. **Don't run as root**: Use USER instruction to run as non-root
4. **Order layers efficiently**: Put frequently changing layers last
5. **Use .dockerignore**: Exclude unnecessary files from build context

### Multi-Stage Build Example

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Runtime stage
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
USER 1000
CMD ["./main"]
```

---

## 2. Choose and Use the Right Workload Resource

Kubernetes provides several workload resources, each designed for specific use cases.

### Deployment

Use for stateless applications that need scaling and rolling updates.

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
        image: nginx:1.25
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

**Imperative Commands:**
```bash
# Create a deployment
kubectl create deployment nginx --image=nginx:1.25 --replicas=3

# Generate YAML without creating
kubectl create deployment nginx --image=nginx:1.25 --dry-run=client -o yaml > deployment.yaml

# Scale a deployment
kubectl scale deployment nginx --replicas=5

# Update image
kubectl set image deployment/nginx nginx=nginx:1.26
```

### ReplicaSet

Maintains a stable set of replica Pods. Typically managed by Deployments.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

**Note:** Use Deployments instead of ReplicaSets directly for declarative updates.

### StatefulSet

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
        image: mysql:8.0
        ports:
        - containerPort: 3306
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

**Key Characteristics:**
- Stable, unique network identifiers (pod-0, pod-1, pod-2)
- Stable, persistent storage per Pod
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

### DaemonSet

Ensures a copy of a Pod runs on all (or some) nodes.

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
      containers:
      - name: fluentd
        image: fluentd:v1.14
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

**Use Cases:**
- Log collectors (Fluentd, Filebeat)
- Node monitoring agents (Prometheus Node Exporter)
- Cluster storage daemons
- Network plugins (CNI)

### Job

For one-time tasks that run to completion.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  completions: 5        # Total successful completions needed
  parallelism: 2        # Pods running in parallel
  backoffLimit: 4       # Retries before marking failed
  activeDeadlineSeconds: 100  # Max duration
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never  # Required: Never or OnFailure
```

**Imperative Commands:**
```bash
# Create a job
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'

# View job status
kubectl get jobs

# View pods created by job
kubectl get pods --selector=job-name=pi
```

### CronJob

For scheduled, recurring tasks.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # Every day at 2 AM
  concurrencyPolicy: Forbid  # Allow, Forbid, or Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:v1
            command: ["/bin/sh", "-c", "backup.sh"]
          restartPolicy: OnFailure
```

**Cron Schedule Format:**
```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday = 0)
│ │ │ │ │
* * * * *
```

**Common Schedules:**
- `*/5 * * * *` - Every 5 minutes
- `0 * * * *` - Every hour
- `0 0 * * *` - Every day at midnight
- `0 0 * * 0` - Every Sunday at midnight
- `0 0 1 * *` - First day of every month

**Imperative Commands:**
```bash
# Create a cronjob
kubectl create cronjob backup --image=backup:v1 --schedule="0 2 * * *" -- /bin/sh -c 'backup.sh'

# Manually trigger a job from cronjob
kubectl create job --from=cronjob/backup backup-manual-001
```

### Workload Selection Guide

| Workload Type | Use Case | State | Scaling |
|---------------|----------|-------|---------|
| Deployment | Web apps, APIs | Stateless | Horizontal |
| StatefulSet | Databases, message queues | Stateful | Ordered |
| DaemonSet | Node agents, log collectors | Per-node | One per node |
| Job | Batch processing, migrations | One-time | Parallel |
| CronJob | Scheduled backups, reports | Recurring | Per schedule |

---

## 3. Multi-Container Pod Design Patterns

Pods can contain multiple containers that share the same network namespace and storage volumes. Several patterns exist for organizing these containers.

### Init Containers

Init containers run to completion before app containers start. Used for setup tasks.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.36
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting; sleep 2; done']
  - name: init-mydb
    image: busybox:1.36
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting; sleep 2; done']
  containers:
  - name: myapp
    image: myapp:v1
```

**Key Characteristics:**
- Run sequentially, one at a time
- Must complete successfully before app containers start
- Can use different images than app containers
- Don't support probes (lifecycle completion is the check)

**Use Cases:**
- Wait for dependent services
- Clone git repos
- Generate configuration files
- Register with service discovery
- Database migrations

### Sidecar Pattern

A container that extends or enhances the main container's functionality.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-shipper
    image: fluentd
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
      readOnly: true
  volumes:
  - name: logs
    emptyDir: {}
```

**Native Sidecar Containers (Kubernetes 1.28+):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
  - name: log-agent
    image: log-agent:v1
    restartPolicy: Always  # Makes it a sidecar
  containers:
  - name: myapp
    image: myapp:v1
```

**Use Cases:**
- Log shipping
- Monitoring agents
- Service mesh proxies (Envoy, Istio)
- Configuration reloaders

### Ambassador Pattern

A container that proxies network connections to/from the main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: DB_HOST
      value: "localhost"  # Connects to ambassador
    - name: DB_PORT
      value: "5432"
  - name: ambassador
    image: haproxy:2.8
    ports:
    - containerPort: 5432
    volumeMounts:
    - name: config
      mountPath: /usr/local/etc/haproxy
  volumes:
  - name: config
    configMap:
      name: haproxy-config
```

**Use Cases:**
- Database connection pooling
- Rate limiting
- Circuit breaking
- Protocol translation

### Adapter Pattern

A container that transforms or normalizes the main container's output.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
spec:
  containers:
  - name: app
    image: legacy-app:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  - name: adapter
    image: log-transformer:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
    - name: transformed
      mountPath: /var/log/transformed
  volumes:
  - name: logs
    emptyDir: {}
  - name: transformed
    emptyDir: {}
```

**Use Cases:**
- Log format transformation
- Metrics format conversion (custom to Prometheus)
- Data format normalization

---

## 4. Utilize Persistent and Ephemeral Volumes

### Volume Types Overview

Kubernetes supports many volume types for different storage needs.

### Ephemeral Volumes

**emptyDir**: Temporary storage that exists for the Pod's lifetime.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - name: test-container
    image: nginx
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
    - name: memory-volume
      mountPath: /ramdisk
  volumes:
  - name: cache-volume
    emptyDir: {}
  - name: memory-volume
    emptyDir:
      medium: Memory  # RAM-backed for speed
      sizeLimit: 100Mi
```

**Use Cases:**
- Scratch space for sorting algorithms
- Checkpointing during long computations
- Sharing files between containers in a Pod

### Persistent Volumes (PV) and Persistent Volume Claims (PVC)

**PersistentVolume (PV)**: Cluster-wide storage resource provisioned by admin.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

**PersistentVolumeClaim (PVC)**: User's request for storage.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

**Using PVC in a Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-example
```

### Access Modes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| ReadWriteOnce | RWO | Mount read-write by single node |
| ReadOnlyMany | ROX | Mount read-only by many nodes |
| ReadWriteMany | RWX | Mount read-write by many nodes |
| ReadWriteOncePod | RWOP | Mount read-write by single Pod |

### Reclaim Policies

| Policy | Description |
|--------|-------------|
| Retain | Manual reclamation required |
| Delete | Volume deleted when PVC released |
| Recycle | Basic scrub (rm -rf), deprecated |

### StorageClass for Dynamic Provisioning

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

**PVC with StorageClass:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast  # References StorageClass
```

### ConfigMap and Secret Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: config.yaml
        path: app-config.yaml
  - name: secret-volume
    secret:
      secretName: app-secrets
      defaultMode: 0400
```

### Projected Volumes

Combine multiple volume sources into a single directory.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: all-in-one
      mountPath: /projected-volume
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
          - key: username
            path: my-group/username
      - configMap:
          name: myconfigmap
          items:
          - key: config
            path: my-group/config
      - downwardAPI:
          items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
```

---

## 5. Essential kubectl Commands for This Domain

```bash
# Image and Container Commands
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment nginx --image=nginx --replicas=3

# Workload Management
kubectl create deployment myapp --image=myapp:v1
kubectl scale deployment myapp --replicas=5
kubectl set image deployment/myapp myapp=myapp:v2
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp

# Jobs and CronJobs
kubectl create job myjob --image=busybox -- echo "hello"
kubectl create cronjob mycron --image=busybox --schedule="*/5 * * * *" -- date
kubectl create job --from=cronjob/mycron manual-run

# Volume and Storage
kubectl get pv
kubectl get pvc
kubectl get storageclass
kubectl describe pvc mypvc

# Pod Inspection
kubectl get pods -o wide
kubectl describe pod mypod
kubectl logs mypod -c mycontainer
kubectl exec -it mypod -- /bin/sh

# Generate YAML quickly
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
kubectl create job test --image=busybox --dry-run=client -o yaml -- echo hello
```

---

## Exam Tips for This Domain

1. **Master imperative commands**: Use `--dry-run=client -o yaml` to quickly generate manifests
2. **Know when to use each workload**: Deployment vs StatefulSet vs DaemonSet vs Job
3. **Understand init containers**: They run before app containers and must complete successfully
4. **Practice multi-container Pods**: Know sidecar, ambassador, and adapter patterns
5. **Understand volume lifecycle**: emptyDir dies with Pod, PVC persists beyond Pod
6. **Know access modes**: RWO, ROX, RWX and when to use each
7. **Practice CronJob schedules**: Memorize common cron patterns
8. **Use kubectl explain**: `kubectl explain pod.spec.volumes` for quick reference

---

## Practice Exercises

1. Create a Deployment with 3 replicas running nginx:1.25, then update to nginx:1.26 using rolling update
2. Create a CronJob that runs every 5 minutes and logs the current date
3. Create a Pod with an init container that waits for a service, then starts the main container
4. Create a Pod with two containers sharing an emptyDir volume
5. Create a PVC requesting 1Gi of storage, then create a Pod that mounts it
6. Create a Job that computes pi to 2000 decimal places with parallelism of 3
7. Build a multi-stage Dockerfile for a Go application that produces a minimal runtime image