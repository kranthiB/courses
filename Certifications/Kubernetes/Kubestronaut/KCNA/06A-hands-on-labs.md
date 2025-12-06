# KCNA Hands-On Lab Exercises

## Prerequisites

Before starting, you need access to a Kubernetes cluster. Choose ONE of these options:

### Option 1: Minikube (Recommended for Beginners)
```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start

# Verify
kubectl cluster-info
```

### Option 2: kind (Kubernetes in Docker)
```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster

# Verify
kubectl cluster-info
```

### Option 3: Cloud Playgrounds (No Installation)
- **Killercoda**: https://killercoda.com/playgrounds/scenario/kubernetes
- **Play with Kubernetes**: https://labs.play-with-k8s.com/

---

# LAB 1: Kubernetes Fundamentals

## Exercise 1.1: Explore Cluster Components

**Objective**: Understand the cluster architecture

```bash
# View cluster info
kubectl cluster-info

# List all nodes
kubectl get nodes

# Get detailed node information
kubectl describe node <node-name>

# View all namespaces
kubectl get namespaces

# View control plane components (in kube-system)
kubectl get pods -n kube-system

# Identify which pods are which components
kubectl get pods -n kube-system -o wide
```

**Questions to answer**:
1. How many nodes does your cluster have?
2. What container runtime is being used?
3. Which control plane components can you identify?

---

## Exercise 1.2: Working with Pods

**Objective**: Create, inspect, and delete Pods

```bash
# Create a simple Pod imperatively
kubectl run nginx-pod --image=nginx:1.21

# Check Pod status
kubectl get pods
kubectl get pods -o wide

# View Pod details
kubectl describe pod nginx-pod

# View Pod logs
kubectl logs nginx-pod

# Execute command in Pod
kubectl exec -it nginx-pod -- /bin/bash
# Inside the pod:
# cat /etc/nginx/nginx.conf
# exit

# Delete the Pod
kubectl delete pod nginx-pod
```

**Now create a Pod declaratively:**

```bash
# Create pod.yaml
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp-container
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
EOF

# Apply the manifest
kubectl apply -f pod.yaml

# Verify
kubectl get pods --show-labels

# Clean up
kubectl delete -f pod.yaml
```

---

## Exercise 1.3: Working with Deployments

**Objective**: Create and manage Deployments

```bash
# Create a Deployment
kubectl create deployment web-app --image=nginx:1.21 --replicas=3

# Check status
kubectl get deployments
kubectl get replicasets
kubectl get pods

# Scale the Deployment
kubectl scale deployment web-app --replicas=5
kubectl get pods -w  # Watch pods scaling (Ctrl+C to exit)

# Update the image (Rolling Update)
kubectl set image deployment/web-app nginx=nginx:1.22

# Watch the rollout
kubectl rollout status deployment/web-app

# View rollout history
kubectl rollout history deployment/web-app

# Rollback to previous version
kubectl rollout undo deployment/web-app

# Clean up
kubectl delete deployment web-app
```

**Declarative Deployment:**

```bash
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:1.21
        ports:
        - containerPort: 80
EOF

kubectl apply -f deployment.yaml
kubectl get all -l app=myapp
kubectl delete -f deployment.yaml
```

---

## Exercise 1.4: Working with Services

**Objective**: Expose applications using Services

```bash
# Create a deployment first
kubectl create deployment web --image=nginx --replicas=3

# Expose as ClusterIP (default)
kubectl expose deployment web --port=80 --target-port=80

# Check the service
kubectl get services
kubectl describe service web

# Test the service (from within cluster)
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- http://web

# Expose as NodePort
kubectl expose deployment web --port=80 --type=NodePort --name=web-nodeport

# Get the NodePort
kubectl get service web-nodeport

# On Minikube, get the URL
minikube service web-nodeport --url

# Clean up
kubectl delete deployment web
kubectl delete service web web-nodeport
```

---

## Exercise 1.5: ConfigMaps and Secrets

**Objective**: Manage configuration and sensitive data

```bash
# Create ConfigMap from literal values
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=mysql.default.svc \
  --from-literal=LOG_LEVEL=INFO

# View ConfigMap
kubectl get configmap app-config -o yaml

# Create Secret
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=API_KEY=abc123xyz

# View Secret (base64 encoded)
kubectl get secret app-secret -o yaml

# Decode secret value
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Use in a Pod
cat <<EOF > pod-with-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD
EOF

kubectl apply -f pod-with-config.yaml

# Verify environment variables
kubectl exec app-pod -- env | grep -E "DATABASE_HOST|DB_PASSWORD"

# Clean up
kubectl delete pod app-pod
kubectl delete configmap app-config
kubectl delete secret app-secret
```

---

## Exercise 1.6: Namespaces

**Objective**: Work with namespaces for resource isolation

```bash
# List namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace development
kubectl create namespace production

# Create resources in specific namespace
kubectl create deployment dev-app --image=nginx -n development
kubectl create deployment prod-app --image=nginx -n production

# List pods in different namespaces
kubectl get pods -n development
kubectl get pods -n production
kubectl get pods -A  # All namespaces

# Set default namespace for current context
kubectl config set-context --current --namespace=development

# Verify
kubectl config view --minify | grep namespace

# Reset to default
kubectl config set-context --current --namespace=default

# Clean up
kubectl delete namespace development production
```

---

# LAB 2: Container Orchestration

## Exercise 2.1: DaemonSets

**Objective**: Deploy a Pod to every node

```bash
cat <<EOF > daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logging-agent
spec:
  selector:
    matchLabels:
      app: logging-agent
  template:
    metadata:
      labels:
        app: logging-agent
    spec:
      containers:
      - name: agent
        image: busybox
        command: ['sh', '-c', 'while true; do echo "Collecting logs from \$(hostname)"; sleep 60; done']
EOF

kubectl apply -f daemonset.yaml

# Check - should have one pod per node
kubectl get daemonset
kubectl get pods -o wide

# View logs
kubectl logs -l app=logging-agent

# Clean up
kubectl delete -f daemonset.yaml
```

---

## Exercise 2.2: Jobs and CronJobs

**Objective**: Run batch workloads

```bash
# Create a Job
cat <<EOF > job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculator
spec:
  completions: 3
  parallelism: 2
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
EOF

kubectl apply -f job.yaml

# Watch job progress
kubectl get jobs -w

# View completed pods
kubectl get pods -l job-name=pi-calculator

# Create a CronJob
cat <<EOF > cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"  # Every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ['sh', '-c', 'echo "Hello at \$(date)"']
          restartPolicy: OnFailure
EOF

kubectl apply -f cronjob.yaml

# Watch for job creation
kubectl get cronjobs
kubectl get jobs -w

# Clean up
kubectl delete job pi-calculator
kubectl delete cronjob hello-cron
```

---

## Exercise 2.3: Resource Limits and Requests

**Objective**: Configure resource constraints

```bash
cat <<EOF > resource-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: demo
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF

kubectl apply -f resource-demo.yaml

# Check resource allocation
kubectl describe pod resource-demo | grep -A 5 "Limits\|Requests"

# View node resource usage
kubectl top nodes  # Requires metrics-server
kubectl top pods

# Clean up
kubectl delete pod resource-demo
```

---

## Exercise 2.4: Probes (Health Checks)

**Objective**: Configure liveness and readiness probes

```bash
cat <<EOF > probes-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-demo
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
EOF

kubectl apply -f probes-demo.yaml

# Watch pod status
kubectl get pods -w

# Check probe status
kubectl describe pod probes-demo | grep -A 10 "Liveness\|Readiness"

# Clean up
kubectl delete pod probes-demo
```

---

## Exercise 2.5: Network Policies (If CNI supports it)

**Objective**: Control pod-to-pod traffic

```bash
# Create test pods
kubectl create deployment frontend --image=nginx
kubectl create deployment backend --image=nginx
kubectl expose deployment backend --port=80

# Test connectivity (should work)
kubectl exec -it $(kubectl get pod -l app=frontend -o name) -- curl -s --max-time 3 http://backend

# Create Network Policy to deny all ingress
cat <<EOF > deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
EOF

kubectl apply -f deny-all.yaml

# Test again (should fail/timeout)
kubectl exec -it $(kubectl get pod -l app=frontend -o name) -- curl -s --max-time 3 http://backend

# Allow traffic from frontend only
cat <<EOF > allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
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
EOF

kubectl apply -f allow-frontend.yaml

# Test again (should work now)
kubectl exec -it $(kubectl get pod -l app=frontend -o name) -- curl -s --max-time 3 http://backend

# Clean up
kubectl delete deployment frontend backend
kubectl delete service backend
kubectl delete networkpolicy deny-all-ingress allow-frontend
```

---

# LAB 3: Helm and Kustomize

## Exercise 3.1: Helm Basics

**Objective**: Use Helm to deploy applications

```bash
# Install Helm (if not installed)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo nginx

# Install a chart
helm install my-nginx bitnami/nginx

# List releases
helm list

# Get release status
helm status my-nginx

# View generated manifests
helm get manifest my-nginx

# Upgrade with custom values
helm upgrade my-nginx bitnami/nginx --set replicaCount=3

# Rollback
helm rollback my-nginx 1

# Uninstall
helm uninstall my-nginx
```

---

## Exercise 3.2: Create Your Own Helm Chart

```bash
# Create a new chart
helm create myapp

# Explore the structure
ls -la myapp/
cat myapp/Chart.yaml
cat myapp/values.yaml

# Install your chart
helm install my-release ./myapp

# List resources
kubectl get all -l app.kubernetes.io/instance=my-release

# Clean up
helm uninstall my-release
rm -rf myapp
```

---

## Exercise 3.3: Kustomize

**Objective**: Use Kustomize for environment-specific configurations

```bash
# Create directory structure
mkdir -p myapp/{base,overlays/dev,overlays/prod}

# Create base deployment
cat <<EOF > myapp/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
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
        image: nginx:1.21
        ports:
        - containerPort: 80
EOF

# Create base kustomization
cat <<EOF > myapp/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
EOF

# Create dev overlay
cat <<EOF > myapp/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namePrefix: dev-
replicas:
- name: myapp
  count: 1
EOF

# Create prod overlay
cat <<EOF > myapp/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namePrefix: prod-
replicas:
- name: myapp
  count: 5
images:
- name: nginx
  newTag: "1.22"
EOF

# Preview dev
kubectl kustomize myapp/overlays/dev

# Preview prod
kubectl kustomize myapp/overlays/prod

# Apply dev
kubectl apply -k myapp/overlays/dev

# Check
kubectl get deployment dev-myapp

# Apply prod
kubectl apply -k myapp/overlays/prod

# Check
kubectl get deployment prod-myapp

# Clean up
kubectl delete -k myapp/overlays/dev
kubectl delete -k myapp/overlays/prod
rm -rf myapp
```

---

# LAB 4: Observability

## Exercise 4.1: View Logs and Events

```bash
# Create a deployment
kubectl create deployment web --image=nginx

# View pod logs
kubectl logs -l app=web

# Stream logs
kubectl logs -l app=web -f

# View cluster events
kubectl get events --sort-by='.lastTimestamp'

# Watch events
kubectl get events -w

# Clean up
kubectl delete deployment web
```

---

## Exercise 4.2: Resource Metrics (Requires metrics-server)

```bash
# Check if metrics-server is installed
kubectl get deployment metrics-server -n kube-system

# On Minikube, enable metrics-server
minikube addons enable metrics-server

# View node metrics
kubectl top nodes

# View pod metrics
kubectl top pods -A

# View pod metrics sorted by CPU
kubectl top pods -A --sort-by=cpu
```

---

# LAB 5: Troubleshooting Practice

## Exercise 5.1: Debug a Failing Pod

```bash
# Create a pod with an error
cat <<EOF > broken-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
spec:
  containers:
  - name: app
    image: nginx:nonexistent
EOF

kubectl apply -f broken-pod.yaml

# Practice debugging
kubectl get pods
kubectl describe pod broken-pod
kubectl get events --field-selector involvedObject.name=broken-pod

# Fix it
kubectl set image pod/broken-pod app=nginx:latest

# Or delete and recreate with correct image
kubectl delete pod broken-pod
```

---

## Exercise 5.2: Debug a Service

```bash
# Create deployment and wrong service
kubectl create deployment web --image=nginx

cat <<EOF > wrong-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: wrong-label  # Wrong selector!
  ports:
  - port: 80
EOF

kubectl apply -f wrong-service.yaml

# Debug
kubectl get endpoints web-service  # Will be empty!
kubectl describe service web-service
kubectl get pods --show-labels

# Fix by updating selector
kubectl patch service web-service -p '{"spec":{"selector":{"app":"web"}}}'

# Verify
kubectl get endpoints web-service  # Should have endpoints now

# Clean up
kubectl delete deployment web
kubectl delete service web-service
```

---

## Lab Completion Checklist

- [ ] Created and managed Pods
- [ ] Created and scaled Deployments
- [ ] Exposed services with different types
- [ ] Used ConfigMaps and Secrets
- [ ] Worked with namespaces
- [ ] Created DaemonSets
- [ ] Created Jobs and CronJobs
- [ ] Configured resource limits
- [ ] Set up health probes
- [ ] Created Network Policies
- [ ] Used Helm to deploy applications
- [ ] Used Kustomize for overlays
- [ ] Viewed logs and events
- [ ] Debugged failing resources

**Complete all labs before taking the exam!**