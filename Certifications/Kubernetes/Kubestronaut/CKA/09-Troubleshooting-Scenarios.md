# CKA Troubleshooting Scenarios

## Purpose

These scenarios simulate real-world Kubernetes issues you might encounter on the CKA exam. Each scenario includes:
- Problem description
- Symptoms
- Investigation steps
- Solution

Practice diagnosing issues WITHOUT looking at solutions first.

---

## Scenario 1: Pod Stuck in Pending State

### Problem
A pod is stuck in `Pending` state and never gets scheduled.

### Setup
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "100Gi"
        cpu: "100"
EOF
```

### Symptoms
```bash
kubectl get pod pending-pod
# NAME          READY   STATUS    RESTARTS   AGE
# pending-pod   0/1     Pending   0          5m
```

### Investigation Steps
```bash
# Step 1: Describe the pod
kubectl describe pod pending-pod

# Step 2: Check events at the bottom
# Look for: "0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory"

# Step 3: Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes
```

### Root Cause
Resource requests exceed available cluster capacity.

### Solution
```bash
# Fix by reducing resource requests
kubectl delete pod pending-pod

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
EOF
```

---

## Scenario 2: Pod in CrashLoopBackOff

### Problem
A pod keeps crashing and restarting.

### Setup
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'exit 1']
EOF
```

### Symptoms
```bash
kubectl get pod crash-pod
# NAME        READY   STATUS             RESTARTS   AGE
# crash-pod   0/1     CrashLoopBackOff   5          3m
```

### Investigation Steps
```bash
# Step 1: Check pod status
kubectl describe pod crash-pod

# Step 2: Check container logs
kubectl logs crash-pod
kubectl logs crash-pod --previous

# Step 3: Look at exit code
kubectl get pod crash-pod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

### Root Cause
Container command exits with non-zero exit code (error).

### Solution
Fix the application or command to not exit with error.

---

## Scenario 3: Service Has No Endpoints

### Problem
A service exists but has no endpoints, so it's not routing traffic.

### Setup
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  labels:
    app: backend
    version: v1
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
    version: v2
  ports:
  - port: 80
    targetPort: 80
EOF
```

### Symptoms
```bash
kubectl get endpoints backend-svc
# NAME          ENDPOINTS   AGE
# backend-svc   <none>      1m

kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- --timeout=2 backend-svc
# wget: download timed out
```

### Investigation Steps
```bash
# Step 1: Check service selector
kubectl get svc backend-svc -o yaml | grep -A 5 selector

# Step 2: Check pod labels
kubectl get pods --show-labels

# Step 3: Compare selectors with labels
# Service selector: app=backend, version=v2
# Pod labels: app=backend, version=v1
```

### Root Cause
Service selector doesn't match pod labels (version mismatch).

### Solution
```bash
# Fix by updating service selector or pod labels
kubectl patch svc backend-svc -p '{"spec":{"selector":{"app":"backend","version":"v1"}}}'

# Verify
kubectl get endpoints backend-svc
```

---

## Scenario 4: DNS Not Resolving

### Problem
Pods cannot resolve service names via DNS.

### Setup
```bash
# Simulate by scaling down CoreDNS (DON'T DO IN PRODUCTION)
kubectl scale deployment coredns -n kube-system --replicas=0
```

### Symptoms
```bash
kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes
# ;; connection timed out; no servers could be reached
```

### Investigation Steps
```bash
# Step 1: Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Step 2: Check CoreDNS deployment
kubectl get deployment coredns -n kube-system

# Step 3: Check CoreDNS logs (if pods exist)
kubectl logs -n kube-system -l k8s-app=kube-dns

# Step 4: Check DNS service
kubectl get svc -n kube-system kube-dns
```

### Root Cause
CoreDNS deployment scaled to 0 replicas.

### Solution
```bash
kubectl scale deployment coredns -n kube-system --replicas=2

# Verify
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes
```

---

## Scenario 5: ImagePullBackOff

### Problem
Pod cannot pull container image.

### Setup
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: bad-image-pod
spec:
  containers:
  - name: app
    image: nginx:nonexistent-tag-12345
EOF
```

### Symptoms
```bash
kubectl get pod bad-image-pod
# NAME            READY   STATUS             RESTARTS   AGE
# bad-image-pod   0/1     ImagePullBackOff   0          2m
```

### Investigation Steps
```bash
# Step 1: Describe pod
kubectl describe pod bad-image-pod

# Step 2: Look at events
# "Failed to pull image "nginx:nonexistent-tag-12345": rpc error..."

# Step 3: Check image name
kubectl get pod bad-image-pod -o jsonpath='{.spec.containers[0].image}'
```

### Root Cause
Image tag doesn't exist in registry.

### Solution
```bash
kubectl set image pod/bad-image-pod app=nginx:latest
# Or delete and recreate with correct image
```

---

## Scenario 6: Node Not Ready

### Problem
A worker node is showing NotReady status.

### Simulated Symptoms
```bash
kubectl get nodes
# NAME       STATUS     ROLES           AGE   VERSION
# master     Ready      control-plane   10d   v1.30.0
# worker-1   NotReady   <none>          10d   v1.30.0
```

### Investigation Steps
```bash
# Step 1: Describe node
kubectl describe node worker-1
# Look for Conditions section

# Step 2: SSH to node and check kubelet
ssh worker-1
systemctl status kubelet
journalctl -u kubelet -n 50

# Step 3: Check container runtime
systemctl status containerd
crictl ps

# Step 4: Check disk space
df -h

# Step 5: Check memory
free -m

# Step 6: Check certificates
ls -la /var/lib/kubelet/pki/
```

### Common Root Causes
1. Kubelet not running
2. Container runtime not running
3. Network plugin issues
4. Disk pressure
5. Memory pressure
6. Certificate expiration

### Solutions
```bash
# Restart kubelet
systemctl restart kubelet

# Restart container runtime
systemctl restart containerd

# Clear disk space if needed
# Check and fix network plugin
```

---

## Scenario 7: Deployment Not Updating

### Problem
Deployment rollout seems stuck.

### Setup
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stuck-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: stuck
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: stuck
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "50"
EOF
```

### Symptoms
```bash
kubectl rollout status deployment/stuck-deploy
# Waiting for deployment "stuck-deploy" rollout to finish...
# (Never completes)
```

### Investigation Steps
```bash
# Step 1: Check deployment status
kubectl get deployment stuck-deploy

# Step 2: Check replicaset
kubectl get rs -l app=stuck

# Step 3: Check pods
kubectl get pods -l app=stuck

# Step 4: Describe new pods
kubectl describe pod <new-pod-name>
# Look for scheduling issues
```

### Root Cause
New pods cannot be scheduled due to excessive resource requests.

### Solution
```bash
kubectl patch deployment stuck-deploy -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","resources":{"requests":{"cpu":"50m"}}}]}}}}'
```

---

## Scenario 8: PVC Stuck in Pending

### Problem
PVC is not binding to a PV.

### Setup
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pending-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: available-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /tmp/data
EOF
```

### Symptoms
```bash
kubectl get pvc pending-pvc
# NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# pending-pvc   Pending                                      manual         1m
```

### Investigation Steps
```bash
# Step 1: Describe PVC
kubectl describe pvc pending-pvc
# Events show: "no persistent volumes available for this claim"

# Step 2: Check available PVs
kubectl get pv

# Step 3: Compare PVC requirements with PV capabilities
# PVC wants: 10Gi, ReadWriteMany
# PV has: 5Gi, ReadWriteOnce
```

### Root Causes
1. Capacity mismatch (PVC requests more than PV offers)
2. Access mode mismatch (PVC wants RWX, PV offers RWO)

### Solution
Either create a matching PV or update the PVC requirements.

---

## Scenario 9: Network Policy Blocking Traffic

### Problem
Pods can't communicate after NetworkPolicy is applied.

### Setup
```bash
kubectl create namespace netpol-test

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: netpol-test
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: netpol-test
  labels:
    app: client
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sleep', '3600']
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

### Symptoms
```bash
kubectl exec -n netpol-test client -- wget -qO- --timeout=2 web
# wget: download timed out
```

### Investigation Steps
```bash
# Step 1: List network policies
kubectl get networkpolicy -n netpol-test

# Step 2: Describe policy
kubectl describe networkpolicy deny-all -n netpol-test

# Step 3: Check if policy applies to pods
# podSelector: {} means all pods
```

### Root Cause
NetworkPolicy denies all ingress and egress traffic.

### Solution
```bash
# Add policy to allow specific traffic
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-web
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 80
EOF
```

---

## Scenario 10: Control Plane Component Down

### Problem
API server is not responding.

### Simulated Symptoms
```bash
kubectl get nodes
# The connection to the server was refused
```

### Investigation Steps
```bash
# Step 1: Check if this is on control plane node
# SSH to control plane if needed

# Step 2: Check control plane pods (as static pods)
crictl ps | grep kube-apiserver
crictl ps | grep etcd

# Step 3: Check static pod manifests
ls -la /etc/kubernetes/manifests/

# Step 4: Check kubelet (which manages static pods)
systemctl status kubelet
journalctl -u kubelet -n 100

# Step 5: Check for syntax errors in manifests
cat /etc/kubernetes/manifests/kube-apiserver.yaml | head -50
```

### Common Root Causes
1. Syntax error in static pod manifest
2. Wrong certificate paths
3. etcd not running
4. Kubelet not running
5. Port already in use

### Solutions
```bash
# Fix manifest syntax
vim /etc/kubernetes/manifests/kube-apiserver.yaml

# Restart kubelet
systemctl restart kubelet

# Check etcd
crictl logs <etcd-container-id>
```

---

## Troubleshooting Cheat Sheet

### Quick Diagnostic Commands

```bash
# Overall cluster health
kubectl cluster-info
kubectl get nodes
kubectl get pods -A

# Events (recent issues)
kubectl get events --sort-by='.lastTimestamp' -A

# Specific resource issues
kubectl describe <resource-type> <name>

# Logs
kubectl logs <pod> [-c container] [--previous]

# Resource usage
kubectl top nodes
kubectl top pods

# Network testing
kubectl run test --image=busybox:1.28 --rm -it -- sh
  # Inside: wget, nslookup, ping, nc

# Node level (SSH to node)
systemctl status kubelet
journalctl -u kubelet
crictl ps
df -h
free -m
```

### Common Issue → Check List

| Issue | First Check | Second Check | Third Check |
|-------|------------|--------------|-------------|
| Pod Pending | `describe pod` (events) | Node resources | PVC status |
| Pod CrashLoop | `logs --previous` | Resource limits | Liveness probe |
| Service no endpoints | Selector match | Pod labels | Pod status |
| DNS not working | CoreDNS pods | CoreDNS logs | kube-dns service |
| Node NotReady | `describe node` | kubelet status | containerd status |
| Deployment stuck | New RS status | New pod events | Resource availability |
| PVC Pending | Matching PV | Access modes | Storage class |

---

## Practice Routine

1. **Create the scenario** (or have someone else create it)
2. **Diagnose WITHOUT looking at solution** (set 5-minute timer)
3. **Document your troubleshooting steps**
4. **Fix the issue**
5. **Verify the fix**
6. **Review the solution** to see if there was a faster path

**Goal:** Solve each scenario type in under 5 minutes.