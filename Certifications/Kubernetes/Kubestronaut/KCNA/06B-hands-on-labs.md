# KCNA Hands-On Labs - Complete Practical Guide

## Essential Labs to Fill the Experience Gap

---

## Lab Environment Setup

```bash
# Option 1: Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start

# Option 2: kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
kind create cluster

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

---

## LAB 1: Kubernetes Core Objects

### Objective
Create and manage Pods, Deployments, Services, ConfigMaps, and Secrets.

### Step 1: Create a Pod
```bash
# Imperative (quick)
kubectl run nginx-pod --image=nginx:latest --port=80

# Declarative (production)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-declarative
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
EOF

# View pods
kubectl get pods
kubectl describe pod nginx-pod
```

### Step 2: Create a Deployment
```bash
# Imperative
kubectl create deployment web-app --image=nginx:1.21 --replicas=3

# Declarative
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
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
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

# View deployment
kubectl get deployments
kubectl describe deployment web-deployment
kubectl get replicasets
kubectl get pods -l app=web
```

### Step 3: Scale and Update
```bash
# Scale
kubectl scale deployment web-deployment --replicas=5
kubectl get pods -w  # Watch scaling

# Update image (rolling update)
kubectl set image deployment/web-deployment nginx=nginx:1.22

# Watch rollout
kubectl rollout status deployment/web-deployment

# Rollback
kubectl rollout undo deployment/web-deployment

# View rollout history
kubectl rollout history deployment/web-deployment
```

### Step 4: Create Services
```bash
# ClusterIP (internal)
kubectl expose deployment web-deployment --port=80 --target-port=80 --type=ClusterIP --name=web-clusterip

# NodePort (external via node)
kubectl expose deployment web-deployment --port=80 --target-port=80 --type=NodePort --name=web-nodeport

# View services
kubectl get services
kubectl describe service web-clusterip

# Test ClusterIP (from within cluster)
kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- http://web-clusterip
```

### Step 5: ConfigMaps and Secrets
```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false

# Create Secret
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=mysecret123

# View them
kubectl get configmaps
kubectl get secrets
kubectl describe configmap app-config

# Use in a Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo \$APP_ENV \$DB_PASSWORD && sleep 3600"]
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD
EOF

kubectl logs config-pod
```

### Step 6: Clean Up
```bash
kubectl delete pod nginx-pod nginx-declarative config-pod
kubectl delete deployment web-app web-deployment
kubectl delete service web-clusterip web-nodeport
kubectl delete configmap app-config
kubectl delete secret app-secret
```

### ✅ Lab 1 Checkpoint
- [ ] Can create Pods imperatively and declaratively
- [ ] Can create and scale Deployments
- [ ] Understand rolling updates and rollbacks
- [ ] Can create different Service types
- [ ] Can use ConfigMaps and Secrets

---

## LAB 2: Namespaces and Resource Management

### Objective
Work with namespaces, resource quotas, and limit ranges.

### Step 1: Create and Use Namespaces
```bash
# Create namespaces
kubectl create namespace development
kubectl create namespace production

# List namespaces
kubectl get namespaces

# Create resources in specific namespace
kubectl run dev-pod --image=nginx -n development
kubectl run prod-pod --image=nginx -n production

# List pods across namespaces
kubectl get pods -A
kubectl get pods -n development
```

### Step 2: Set Default Namespace
```bash
# View current context
kubectl config current-context

# Set default namespace for context
kubectl config set-context --current --namespace=development

# Now commands default to development namespace
kubectl get pods  # Shows development pods

# Reset to default
kubectl config set-context --current --namespace=default
```

### Step 3: Resource Quotas
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "10"
    services: "5"
EOF

# View quota
kubectl get resourcequota -n development
kubectl describe resourcequota dev-quota -n development
```

### Step 4: Limit Ranges
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
    type: Container
EOF

# View limit range
kubectl describe limitrange dev-limits -n development

# Create pod (will get default limits)
kubectl run limited-pod --image=nginx -n development
kubectl describe pod limited-pod -n development | grep -A 10 "Limits"
```

### Step 5: Clean Up
```bash
kubectl delete namespace development production
```

### ✅ Lab 2 Checkpoint
- [ ] Can create and switch namespaces
- [ ] Understand ResourceQuota purpose
- [ ] Understand LimitRange defaults

---

## LAB 3: Storage (PV, PVC, StorageClass)

### Objective
Work with persistent storage in Kubernetes.

### Step 1: Create a PersistentVolume
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/my-pv-data
EOF

kubectl get pv
```

### Step 2: Create a PersistentVolumeClaim
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF

kubectl get pvc
kubectl get pv  # Should show Bound status
```

### Step 3: Use PVC in a Pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'Hello from PV!' > /data/test.txt && cat /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-pvc
EOF

kubectl logs storage-pod
```

### Step 4: View StorageClasses
```bash
kubectl get storageclasses
kubectl describe storageclass standard  # or whatever exists in your cluster
```

### Step 5: Clean Up
```bash
kubectl delete pod storage-pod
kubectl delete pvc my-pvc
kubectl delete pv my-pv
```

### ✅ Lab 3 Checkpoint
- [ ] Understand PV vs PVC
- [ ] Know access modes (RWO, RWX, ROX)
- [ ] Understand StorageClass purpose

---

## LAB 4: Kubernetes Networking

### Objective
Understand service discovery, DNS, and network basics.

### Step 1: Deploy Backend and Frontend
```bash
# Create namespace
kubectl create namespace networking-lab

# Backend deployment
kubectl create deployment backend --image=nginx -n networking-lab
kubectl expose deployment backend --port=80 -n networking-lab

# Frontend deployment
kubectl create deployment frontend --image=busybox -n networking-lab -- sleep 3600
kubectl scale deployment frontend --replicas=1 -n networking-lab
```

### Step 2: Test DNS Resolution
```bash
# Get frontend pod name
FRONTEND_POD=$(kubectl get pods -n networking-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# Test DNS resolution
kubectl exec -n networking-lab $FRONTEND_POD -- nslookup backend
kubectl exec -n networking-lab $FRONTEND_POD -- nslookup backend.networking-lab.svc.cluster.local

# Test connectivity
kubectl exec -n networking-lab $FRONTEND_POD -- wget -qO- http://backend
```

### Step 3: Explore Service DNS
```bash
# Full DNS format
# <service>.<namespace>.svc.cluster.local

# These all work from within the namespace:
kubectl exec -n networking-lab $FRONTEND_POD -- wget -qO- http://backend
kubectl exec -n networking-lab $FRONTEND_POD -- wget -qO- http://backend.networking-lab
kubectl exec -n networking-lab $FRONTEND_POD -- wget -qO- http://backend.networking-lab.svc.cluster.local
```

### Step 4: View Endpoints
```bash
kubectl get endpoints -n networking-lab
kubectl describe endpoints backend -n networking-lab
```

### Step 5: Create Headless Service
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
  namespace: networking-lab
spec:
  clusterIP: None  # This makes it headless
  selector:
    app: backend
  ports:
  - port: 80
EOF

# DNS returns pod IPs directly
kubectl exec -n networking-lab $FRONTEND_POD -- nslookup backend-headless
```

### Step 6: Clean Up
```bash
kubectl delete namespace networking-lab
```

### ✅ Lab 4 Checkpoint
- [ ] Understand Service DNS format
- [ ] Can test connectivity between pods
- [ ] Know difference between ClusterIP and Headless

---

## LAB 5: Observability (Logs, Metrics, Events)

### Objective
Practice observability commands and concepts.

### Step 1: View Pod Logs
```bash
# Create a pod that generates logs
kubectl run logger --image=busybox -- sh -c "while true; do echo 'Log entry at $(date)'; sleep 5; done"

# Wait for it to start
sleep 10

# View logs
kubectl logs logger
kubectl logs logger --tail=5
kubectl logs logger -f  # Follow (Ctrl+C to exit)

# Previous container logs (if restarted)
kubectl logs logger --previous  # Will error if not restarted
```

### Step 2: Multi-Container Logs
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo 'App log'; sleep 5; done"]
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "while true; do echo 'Sidecar log'; sleep 5; done"]
EOF

sleep 10

# Must specify container
kubectl logs multi-container -c app
kubectl logs multi-container -c sidecar
kubectl logs multi-container --all-containers
```

### Step 3: View Events
```bash
# Cluster events
kubectl get events --sort-by='.lastTimestamp'

# Namespace events
kubectl get events -n default

# Events for specific resource
kubectl describe pod logger | tail -20
```

### Step 4: Resource Usage (if metrics-server available)
```bash
# Check if metrics available
kubectl top nodes 2>/dev/null || echo "Metrics server not installed"
kubectl top pods 2>/dev/null || echo "Metrics server not installed"

# In Minikube, enable metrics
minikube addons enable metrics-server
# Wait a minute, then:
kubectl top nodes
kubectl top pods
```

### Step 5: Debug with Exec
```bash
kubectl exec logger -- ps aux
kubectl exec logger -- cat /etc/os-release
kubectl exec -it logger -- sh  # Interactive shell (type 'exit' to leave)
```

### Step 6: Clean Up
```bash
kubectl delete pod logger multi-container
```

### ✅ Lab 5 Checkpoint
- [ ] Can view and follow logs
- [ ] Can get logs from specific containers
- [ ] Understand kubectl top and events
- [ ] Can exec into containers

---

## LAB 6: Helm Basics

### Objective
Use Helm to deploy applications.

### Step 1: Install Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Step 2: Add a Repository
```bash
# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo nginx
helm search repo bitnami/nginx
```

### Step 3: Install a Chart
```bash
# Install nginx
helm install my-nginx bitnami/nginx

# View what was created
kubectl get all -l app.kubernetes.io/instance=my-nginx

# List releases
helm list
```

### Step 4: Customize with Values
```bash
# View default values
helm show values bitnami/nginx | head -50

# Install with custom values
helm install my-custom-nginx bitnami/nginx \
  --set replicaCount=2 \
  --set service.type=ClusterIP
```

### Step 5: Upgrade and Rollback
```bash
# Upgrade
helm upgrade my-nginx bitnami/nginx --set replicaCount=3

# View history
helm history my-nginx

# Rollback
helm rollback my-nginx 1
```

### Step 6: Uninstall
```bash
helm uninstall my-nginx
helm uninstall my-custom-nginx
```

### ✅ Lab 6 Checkpoint
- [ ] Can add Helm repos
- [ ] Can install charts
- [ ] Understand values customization
- [ ] Can upgrade and rollback

---

## LAB 7: Kubectl Mastery

### Objective
Master essential kubectl commands.

### Step 1: Create Test Resources
```bash
kubectl create deployment test-app --image=nginx --replicas=3
kubectl expose deployment test-app --port=80
```

### Step 2: Output Formats
```bash
# Different output formats
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

### Step 3: Sorting and Filtering
```bash
# Sort by creation time
kubectl get pods --sort-by='.metadata.creationTimestamp'

# Filter by label
kubectl get pods -l app=test-app

# Show labels
kubectl get pods --show-labels
```

### Step 4: Explain Resources
```bash
# Get resource documentation
kubectl explain pods
kubectl explain pods.spec
kubectl explain pods.spec.containers
kubectl explain pods.spec.containers.resources
```

### Step 5: Dry Run and Diff
```bash
# Generate YAML without creating
kubectl create deployment dry-run-test --image=nginx --dry-run=client -o yaml

# Diff before applying
kubectl create deployment diff-test --image=nginx --dry-run=client -o yaml > diff-test.yaml
kubectl diff -f diff-test.yaml
rm diff-test.yaml
```

### Step 6: Context and Config
```bash
# View config
kubectl config view

# View current context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context (if multiple clusters)
# kubectl config use-context <context-name>
```

### Step 7: Clean Up
```bash
kubectl delete deployment test-app
kubectl delete service test-app
```

### ✅ Lab 7 Checkpoint
- [ ] Can use different output formats
- [ ] Can filter and sort resources
- [ ] Understand kubectl explain
- [ ] Can use dry-run

---

## LAB 8: Troubleshooting

### Objective
Practice common troubleshooting scenarios.

### Step 1: Create a Broken Pod
```bash
# Pod with wrong image
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-image
spec:
  containers:
  - name: app
    image: nginx:nonexistent-tag
EOF

# Check status
kubectl get pods
kubectl describe pod broken-image | tail -20
# Look for: ImagePullBackOff or ErrImagePull
```

### Step 2: Pod with Crash Loop
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-loop
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "exit 1"]
EOF

# Check status
kubectl get pods
kubectl describe pod crash-loop | tail -20
kubectl logs crash-loop
# Look for: CrashLoopBackOff
```

### Step 3: Pod with Resource Issues
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-issue
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "1000Gi"  # Impossibly high
EOF

kubectl get pods
kubectl describe pod resource-issue | tail -20
# Look for: Pending, Insufficient memory
```

### Step 4: Debug Running Pods
```bash
# Create a working pod
kubectl run debug-target --image=nginx

# Debug with ephemeral container (K8s 1.23+)
# kubectl debug debug-target -it --image=busybox

# Or exec into the pod
kubectl exec -it debug-target -- bash

# Inside, check:
# - ps aux
# - cat /etc/nginx/nginx.conf
# - curl localhost
# Type 'exit' to leave
```

### Step 5: Check Cluster Health
```bash
# Component status (deprecated but still works)
kubectl get componentstatuses

# Node status
kubectl get nodes
kubectl describe node minikube  # or your node name

# System pods
kubectl get pods -n kube-system
```

### Step 6: Clean Up
```bash
kubectl delete pod broken-image crash-loop resource-issue debug-target
```

### ✅ Lab 8 Checkpoint
- [ ] Can identify ImagePullBackOff
- [ ] Can identify CrashLoopBackOff
- [ ] Can identify resource/scheduling issues
- [ ] Know how to debug running pods

---

## Summary: KCNA Lab Commands Cheatsheet

```bash
# PODS
kubectl run <name> --image=<image>
kubectl get pods [-o wide|yaml|json]
kubectl describe pod <name>
kubectl logs <pod> [-c container] [-f]
kubectl exec -it <pod> -- <command>
kubectl delete pod <name>

# DEPLOYMENTS
kubectl create deployment <name> --image=<image>
kubectl scale deployment <name> --replicas=<n>
kubectl set image deployment/<name> <container>=<image>
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>

# SERVICES
kubectl expose deployment <name> --port=<port> --type=<type>
kubectl get services
kubectl describe service <name>

# CONFIG
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=key=value

# NAMESPACES
kubectl create namespace <name>
kubectl get pods -n <namespace>
kubectl get pods -A  # All namespaces

# DEBUG
kubectl describe <resource> <name>
kubectl logs <pod>
kubectl exec -it <pod> -- sh
kubectl get events

# HELM
helm repo add <name> <url>
helm install <release> <chart>
helm upgrade <release> <chart>
helm rollback <release> <revision>
helm uninstall <release>
```

---

**Complete these labs for hands-on KCNA experience! 🎯**