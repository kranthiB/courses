# CKAD Domain 3: Application Observability and Maintenance (15%)

## Overview

This domain focuses on monitoring, logging, debugging, and maintaining applications running in Kubernetes. You must understand how to implement health checks using probes, access and analyze logs, monitor application metrics, and troubleshoot common issues.

---

## 1. Understand API Deprecations

### Why APIs Get Deprecated

Kubernetes evolves rapidly, and APIs are versioned to allow for improvements while maintaining stability. Understanding API versions helps you maintain compatible manifests.

### API Version Lifecycle

| Stage | Description | Stability |
|-------|-------------|-----------|
| alpha | May be buggy, disabled by default | Not recommended for production |
| beta | Well-tested, enabled by default | Safe for non-critical production |
| stable (v1) | Production-ready | Fully supported |

### Checking for Deprecated APIs

```bash
# Check if resources use deprecated APIs
kubectl get deployments.v1.apps -o yaml | grep apiVersion

# Use kubectl convert plugin (if available)
kubectl convert -f old-deployment.yaml --output-version apps/v1

# Check API versions available in cluster
kubectl api-versions

# Get resources with specific API version
kubectl api-resources --api-group=apps
```

### Common API Migration Examples

**Deployment: extensions/v1beta1 → apps/v1**

```yaml
# OLD (deprecated)
apiVersion: extensions/v1beta1
kind: Deployment

# NEW (current)
apiVersion: apps/v1
kind: Deployment
```

**Ingress: extensions/v1beta1 → networking.k8s.io/v1**

```yaml
# OLD (deprecated)
apiVersion: extensions/v1beta1
kind: Ingress

# NEW (current)
apiVersion: networking.k8s.io/v1
kind: Ingress
```

### Using kubectl explain

```bash
# Get API documentation for a resource
kubectl explain deployment
kubectl explain deployment.spec.strategy
kubectl explain pod.spec.containers.livenessProbe

# See available fields with types
kubectl explain pod.spec --recursive
```

### Checking Events for Deprecation Warnings

```bash
# View cluster events
kubectl get events --all-namespaces | grep -i deprecated

# View events for specific resource
kubectl describe deployment myapp | grep -i deprecated
```

---

## 2. Implement Probes and Health Checks

Kubernetes uses probes to determine the health of containers and make decisions about traffic routing and container restarts.

### Types of Probes

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| **livenessProbe** | Is the container running? | Restart container |
| **readinessProbe** | Is the container ready to serve traffic? | Remove from Service endpoints |
| **startupProbe** | Has the application started? | Kill container (until success) |

### Probe Mechanisms

#### HTTP GET Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 5
      successThreshold: 1
      failureThreshold: 3
```

#### TCP Socket Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    ports:
    - containerPort: 3306
    livenessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 5
      periodSeconds: 5
```

#### Exec Command Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
spec:
  containers:
  - name: app
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

#### gRPC Probe (Kubernetes 1.24+)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-grpc
spec:
  containers:
  - name: app
    image: mygrpc-app:v1
    ports:
    - containerPort: 50051
    livenessProbe:
      grpc:
        port: 50051
        service: health  # Optional service name
      initialDelaySeconds: 10
      periodSeconds: 10
```

### Probe Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `initialDelaySeconds` | Seconds to wait before first probe | 0 |
| `periodSeconds` | How often to perform the probe | 10 |
| `timeoutSeconds` | Seconds to wait for probe response | 1 |
| `successThreshold` | Consecutive successes to be considered healthy | 1 |
| `failureThreshold` | Consecutive failures before taking action | 3 |

### Complete Probe Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
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
        image: myapp:v1
        ports:
        - containerPort: 8080
        
        # Startup probe - runs first, disables other probes until success
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
          # Allows up to 5 minutes for startup (30 * 10 = 300s)
        
        # Liveness probe - restarts container if unhealthy
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0  # Startup probe handles delay
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe - removes from service if not ready
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

### When to Use Each Probe

**Liveness Probe:**
- Application can enter deadlock state
- Application can become unresponsive but process still runs
- Want automatic recovery from hung states

**Readiness Probe:**
- Application needs time to load data/cache
- Application temporarily unable to serve traffic
- Dependent services are temporarily unavailable

**Startup Probe:**
- Application has slow startup time
- Need to protect slow-starting containers from being killed
- Initialization time varies significantly

### Best Practices

1. **Use separate endpoints** for liveness and readiness probes
2. **Keep probes lightweight** - don't include heavy operations
3. **Set appropriate timeouts** - account for network latency
4. **Use startup probes** for slow-starting applications
5. **Don't set liveness probe too aggressive** - avoid unnecessary restarts

---

## 3. Use Built-in CLI Tools to Monitor Applications

### kubectl top - Resource Usage

```bash
# View node resource usage (requires metrics-server)
kubectl top nodes

# View pod resource usage
kubectl top pods
kubectl top pods -n kube-system
kubectl top pods --all-namespaces

# View resource usage for specific pod
kubectl top pod mypod

# Sort by CPU or memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# View container-level metrics
kubectl top pod mypod --containers
```

### kubectl describe - Detailed Information

```bash
# Describe a pod (includes events, status, conditions)
kubectl describe pod mypod

# Describe deployment
kubectl describe deployment mydeployment

# Describe node
kubectl describe node mynode

# Key sections to look for:
# - Status/Phase
# - Conditions
# - Events (at the bottom)
# - Resource requests/limits
```

### kubectl get - Resource Status

```bash
# Get pods with additional info
kubectl get pods -o wide

# Watch pods in real-time
kubectl get pods -w

# Get specific fields
kubectl get pods -o jsonpath='{.items[*].status.phase}'

# Custom columns output
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Get all resources
kubectl get all
kubectl get all -n mynamespace

# Get events sorted by time
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Resource Status Fields

**Pod Conditions:**
```bash
kubectl get pod mypod -o jsonpath='{.status.conditions[*].type}'
# PodScheduled, Initialized, ContainersReady, Ready
```

**Pod Phase:**
- `Pending` - Waiting to be scheduled or image pull
- `Running` - At least one container running
- `Succeeded` - All containers completed successfully
- `Failed` - All containers terminated, at least one failed
- `Unknown` - State cannot be determined

### kubectl cluster-info

```bash
# Display cluster information
kubectl cluster-info

# Dump cluster state for debugging
kubectl cluster-info dump

# Check component status (deprecated but still useful)
kubectl get componentstatuses
```

---

## 4. Utilize Container Logs

### Basic Log Commands

```bash
# View logs for a pod (single container)
kubectl logs mypod

# View logs for specific container in multi-container pod
kubectl logs mypod -c mycontainer

# Follow logs in real-time (like tail -f)
kubectl logs -f mypod

# View last N lines
kubectl logs mypod --tail=100

# View logs from last hour
kubectl logs mypod --since=1h

# View logs since specific time
kubectl logs mypod --since-time=2024-01-15T10:00:00Z

# View previous container logs (after restart)
kubectl logs mypod --previous
kubectl logs mypod -p

# View logs for all containers in pod
kubectl logs mypod --all-containers=true

# View logs for pods with specific label
kubectl logs -l app=myapp

# Follow logs for multiple pods
kubectl logs -f -l app=myapp --all-containers=true
```

### Logs from Workload Resources

```bash
# Logs from deployment (picks one pod)
kubectl logs deployment/mydeployment

# Logs from job
kubectl logs job/myjob

# Logs from daemonset
kubectl logs daemonset/mydaemonset

# Logs from specific pod of a statefulset
kubectl logs statefulset/mystatefulset-0
```

### Combining with grep

```bash
# Filter logs for specific content
kubectl logs mypod | grep ERROR

# Count occurrences
kubectl logs mypod | grep -c "error"

# Get context around matches
kubectl logs mypod | grep -A 5 -B 5 "exception"
```

### Log Output Timestamps

```bash
# Include timestamps
kubectl logs mypod --timestamps=true
```

### Ephemeral Containers for Debugging

```bash
# Add ephemeral container to running pod (Kubernetes 1.25+)
kubectl debug mypod -it --image=busybox --target=mycontainer

# Share process namespace
kubectl debug mypod -it --image=busybox --share-processes
```

---

## 5. Debugging in Kubernetes

### Debugging Pods

**Check Pod Status:**
```bash
kubectl get pod mypod -o yaml
kubectl describe pod mypod
```

**Common Pod Issues:**

| Status | Possible Causes | Debug Commands |
|--------|-----------------|----------------|
| `Pending` | No resources, node selector, taints | `kubectl describe pod` |
| `ImagePullBackOff` | Wrong image name, private registry | `kubectl describe pod` |
| `CrashLoopBackOff` | Application crash, wrong command | `kubectl logs --previous` |
| `ErrImagePull` | Image doesn't exist | `kubectl describe pod` |
| `ContainerCreating` | Volume mount issues, secrets | `kubectl describe pod` |

**Interactive Debugging:**
```bash
# Execute command in running container
kubectl exec mypod -- ls -la /app

# Open interactive shell
kubectl exec -it mypod -- /bin/sh
kubectl exec -it mypod -c mycontainer -- /bin/bash

# Execute in specific container of multi-container pod
kubectl exec -it mypod -c sidecar -- /bin/sh
```

### Debugging with kubectl debug

```bash
# Create debugging pod copying from problematic pod
kubectl debug mypod --copy-to=debug-pod --container=debugger --image=busybox

# Debug a node
kubectl debug node/mynode -it --image=ubuntu

# Create ephemeral container in running pod
kubectl debug -it mypod --image=busybox --target=app
```

### Debugging Services

```bash
# Check service endpoints
kubectl get endpoints myservice

# Check service selector matches pods
kubectl get pods -l app=myapp

# Test service from within cluster
kubectl run test --rm -it --image=busybox -- wget -qO- myservice:80

# Check DNS resolution
kubectl run test --rm -it --image=busybox -- nslookup myservice

# View service details
kubectl describe service myservice
```

### Debugging Network Issues

```bash
# Test connectivity from a pod
kubectl exec mypod -- ping google.com
kubectl exec mypod -- wget -qO- http://myservice
kubectl exec mypod -- curl -v http://myservice:8080

# Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy mypolicy

# DNS debugging
kubectl exec mypod -- cat /etc/resolv.conf
kubectl exec mypod -- nslookup kubernetes.default
```

### Debugging Resource Issues

```bash
# Check resource usage
kubectl top pods
kubectl top nodes

# Describe node for resource pressure
kubectl describe node | grep -A 5 "Conditions"

# Check resource quotas
kubectl describe resourcequota

# Check limit ranges
kubectl describe limitrange
```

### Reading Events

```bash
# Get all events
kubectl get events

# Get events for specific namespace
kubectl get events -n mynamespace

# Sort by timestamp
kubectl get events --sort-by='.lastTimestamp'

# Watch events in real-time
kubectl get events -w

# Get events for specific resource
kubectl get events --field-selector involvedObject.name=mypod
```

### Common Debugging Scenarios

**Application Not Starting:**
```bash
kubectl describe pod mypod  # Check events
kubectl logs mypod --previous  # Check crash logs
kubectl get events | grep mypod  # Related events
```

**Application Keeps Restarting:**
```bash
kubectl logs mypod --previous
kubectl describe pod mypod | grep -A 10 "Last State"
kubectl get pod mypod -o yaml | grep restartCount
```

**Service Not Accessible:**
```bash
kubectl get endpoints myservice  # Are there endpoints?
kubectl get pods -l app=myapp  # Do selector labels match?
kubectl describe service myservice  # Check service config
```

**Pod Stuck in Pending:**
```bash
kubectl describe pod mypod | grep Events -A 20
kubectl describe node | grep -A 5 Allocatable
kubectl get pod mypod -o yaml | grep nodeSelector
```

---

## 6. Essential kubectl Commands for This Domain

```bash
# Probes and Health
kubectl get pods -o jsonpath='{.items[*].status.conditions}'
kubectl describe pod mypod | grep -A 10 "Liveness\|Readiness\|Startup"

# Monitoring
kubectl top nodes
kubectl top pods --all-namespaces
kubectl top pod mypod --containers

# Logs
kubectl logs mypod
kubectl logs mypod -c container --tail=50
kubectl logs -f mypod --since=1h
kubectl logs mypod --previous

# Debugging
kubectl describe pod mypod
kubectl exec -it mypod -- /bin/sh
kubectl debug mypod -it --image=busybox
kubectl get events --sort-by='.lastTimestamp'

# Status Checking
kubectl get pods -o wide
kubectl get pods -w
kubectl get pod mypod -o yaml

# API Information
kubectl api-versions
kubectl api-resources
kubectl explain pod.spec.containers.livenessProbe
```

---

## Exam Tips for This Domain

1. **Know all three probe types**: liveness, readiness, startup and when to use each
2. **Remember probe mechanisms**: httpGet, tcpSocket, exec, grpc
3. **Memorize probe parameters**: initialDelaySeconds, periodSeconds, timeoutSeconds, failureThreshold
4. **Master kubectl logs**: --previous, --tail, --since, -c container, -f
5. **Know kubectl exec**: Essential for debugging running containers
6. **Use kubectl describe**: Always check Events section for issues
7. **Understand pod phases**: Pending, Running, Succeeded, Failed, Unknown
8. **Practice with kubectl top**: Requires metrics-server in cluster

---

## Practice Exercises

1. Create a Pod with all three probes (startup, liveness, readiness) using httpGet
2. Create a Pod that intentionally fails its liveness probe and observe the restart
3. Debug a CrashLoopBackOff pod using kubectl logs --previous
4. Use kubectl exec to verify service connectivity from within a pod
5. Find pods consuming the most CPU using kubectl top
6. Create a multi-container pod and retrieve logs from a specific container
7. Analyze events for a failing deployment and identify the root cause
8. Create a Pod with an exec probe that checks for the existence of a file
9. Debug a pod stuck in Pending state by examining events and node resources
10. Use kubectl debug to add an ephemeral container for troubleshooting