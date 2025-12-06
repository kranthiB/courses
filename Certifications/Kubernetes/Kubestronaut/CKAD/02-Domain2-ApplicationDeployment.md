# CKAD Domain 2: Application Deployment (20%)

## Overview

This domain evaluates your ability to deploy applications on Kubernetes clusters. You must understand how to perform rolling updates and rollbacks, use deployment strategies like blue/green and canary, and leverage package managers like Helm and configuration tools like Kustomize.

---

## 1. Understand Deployments and Perform Rolling Updates

### Deployment Fundamentals

A Deployment provides declarative updates for Pods and ReplicaSets. It manages the lifecycle of your application, handling scaling, updates, and rollbacks.

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
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods over desired during update
      maxUnavailable: 0  # Max unavailable pods during update
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### Deployment Strategies

#### RollingUpdate (Default)

Gradually replaces old pods with new ones, maintaining availability.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Can be absolute number or percentage
      maxUnavailable: 25%  # Can be absolute number or percentage
```

**Parameters:**
- **maxSurge**: Maximum number of pods that can be created over the desired replica count
- **maxUnavailable**: Maximum number of pods that can be unavailable during the update

**Common Configurations:**
```yaml
# Zero-downtime deployment (conservative)
rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0

# Fast deployment (allows some downtime)
rollingUpdate:
  maxSurge: 50%
  maxUnavailable: 50%

# Balanced approach
rollingUpdate:
  maxSurge: 25%
  maxUnavailable: 25%
```

#### Recreate Strategy

Terminates all existing pods before creating new ones. Causes downtime.

```yaml
spec:
  strategy:
    type: Recreate
```

**Use Cases:**
- Application cannot run multiple versions simultaneously
- Database migrations that require exclusive access
- Resource-constrained environments

### Performing Rolling Updates

**Update image using kubectl:**
```bash
# Update the image
kubectl set image deployment/nginx-deployment nginx=nginx:1.25

# Check rollout status
kubectl rollout status deployment/nginx-deployment

# Watch the rollout in real-time
kubectl get pods -w

# View rollout history
kubectl rollout history deployment/nginx-deployment

# See details of a specific revision
kubectl rollout history deployment/nginx-deployment --revision=2
```

**Update using kubectl edit:**
```bash
kubectl edit deployment/nginx-deployment
# Edit the image field and save
```

**Update using kubectl apply:**
```bash
# Modify the YAML file and apply
kubectl apply -f nginx-deployment.yaml
```

### Rollback a Deployment

```bash
# Rollback to previous revision
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Check the rollback status
kubectl rollout status deployment/nginx-deployment
```

### Pause and Resume Rollouts

Useful for making multiple changes before triggering a rollout.

```bash
# Pause the deployment
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi

# Resume to apply all changes at once
kubectl rollout resume deployment/nginx-deployment
```

### Scale a Deployment

```bash
# Manual scaling
kubectl scale deployment/nginx-deployment --replicas=5

# Autoscaling (HPA)
kubectl autoscale deployment/nginx-deployment --min=3 --max=10 --cpu-percent=80
```

### Revision History

Control how many old ReplicaSets are retained:

```yaml
spec:
  revisionHistoryLimit: 10  # Default is 10, set to 0 to disable
```

---

## 2. Blue/Green and Canary Deployment Strategies

Kubernetes doesn't have built-in blue/green or canary support, but you can implement them using labels, selectors, and Services.

### Blue/Green Deployment

Run two identical environments (blue = current, green = new), switch traffic instantly.

**Step 1: Create Blue Deployment (Current Version)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1
        ports:
        - containerPort: 8080
```

**Step 2: Create Service Pointing to Blue**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Points to blue deployment
  ports:
  - port: 80
    targetPort: 8080
```

**Step 3: Deploy Green (New Version)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2
        ports:
        - containerPort: 8080
```

**Step 4: Test Green Environment**

```bash
# Create a temporary service to test green
kubectl expose deployment app-green --name=myapp-green-test --port=80 --target-port=8080

# Test the new version
kubectl run test-pod --rm -it --image=busybox -- wget -qO- myapp-green-test

# Remove test service
kubectl delete service myapp-green-test
```

**Step 5: Switch Traffic to Green**

```bash
# Update the service selector to point to green
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'
```

**Step 6: Cleanup Blue (After Validation)**

```bash
# Delete old deployment when confident
kubectl delete deployment app-blue
```

### Canary Deployment

Gradually shift traffic from old to new version to test with real users.

**Step 1: Existing Stable Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9  # 90% of traffic
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: app
        image: myapp:v1
        ports:
        - containerPort: 8080
```

**Step 2: Create Canary Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1  # 10% of traffic (1 out of 10 total pods)
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: app
        image: myapp:v2
        ports:
        - containerPort: 8080
```

**Step 3: Service Selects Both (Using Common Label)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp  # Matches both stable AND canary
  ports:
  - port: 80
    targetPort: 8080
```

**Step 4: Gradually Increase Canary Traffic**

```bash
# Increase canary replicas
kubectl scale deployment app-canary --replicas=3

# Decrease stable replicas
kubectl scale deployment app-stable --replicas=7

# Continue until fully migrated
kubectl scale deployment app-canary --replicas=10
kubectl scale deployment app-stable --replicas=0

# Delete old deployment
kubectl delete deployment app-stable

# Rename canary to stable (optional)
kubectl patch deployment app-canary -p '{"metadata":{"name":"app-stable"}}'
```

### Traffic Splitting with Labels

| Scenario | Stable Replicas | Canary Replicas | Traffic Split |
|----------|-----------------|-----------------|---------------|
| Initial | 10 | 0 | 100% / 0% |
| Start Canary | 9 | 1 | 90% / 10% |
| Increase | 7 | 3 | 70% / 30% |
| More Traffic | 5 | 5 | 50% / 50% |
| Full Rollout | 0 | 10 | 0% / 100% |

---

## 3. Use Helm Package Manager

Helm is the package manager for Kubernetes. It uses "charts" to define, install, and upgrade applications.

### Helm Concepts

- **Chart**: A Helm package containing Kubernetes resource definitions
- **Repository**: A place where charts are stored and shared
- **Release**: An instance of a chart running in a cluster

### Basic Helm Commands

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repositories
helm repo update

# Search for charts
helm search repo nginx
helm search hub wordpress  # Search Artifact Hub

# Show chart information
helm show chart bitnami/nginx
helm show values bitnami/nginx
helm show readme bitnami/nginx
helm show all bitnami/nginx
```

### Installing Charts

```bash
# Install a chart with default values
helm install my-nginx bitnami/nginx

# Install with custom values file
helm install my-nginx bitnami/nginx -f values.yaml

# Install with inline value overrides
helm install my-nginx bitnami/nginx --set service.type=NodePort

# Install with multiple value overrides
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort \
  --set replicaCount=3 \
  --set image.tag=1.25.0

# Install in specific namespace (creates if doesn't exist)
helm install my-nginx bitnami/nginx -n web --create-namespace

# Install with a specific version
helm install my-nginx bitnami/nginx --version 15.0.0

# Dry run (preview without installing)
helm install my-nginx bitnami/nginx --dry-run

# Generate YAML output
helm template my-nginx bitnami/nginx > nginx-manifests.yaml
```

### Managing Releases

```bash
# List installed releases
helm list
helm list -A  # All namespaces
helm list -n web  # Specific namespace

# Get release status
helm status my-nginx

# Get release values
helm get values my-nginx
helm get values my-nginx --all  # Include defaults

# Get manifests for a release
helm get manifest my-nginx

# Get release history
helm history my-nginx
```

### Upgrading Releases

```bash
# Upgrade with new values
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Upgrade with values file
helm upgrade my-nginx bitnami/nginx -f new-values.yaml

# Upgrade or install if doesn't exist
helm upgrade --install my-nginx bitnami/nginx

# Upgrade with atomic (rollback on failure)
helm upgrade my-nginx bitnami/nginx --atomic --timeout 5m

# Reuse existing values and add new ones
helm upgrade my-nginx bitnami/nginx --reuse-values --set newKey=newValue
```

### Rolling Back

```bash
# Rollback to previous revision
helm rollback my-nginx

# Rollback to specific revision
helm rollback my-nginx 2

# View history to find revision number
helm history my-nginx
```

### Uninstalling Releases

```bash
# Uninstall a release
helm uninstall my-nginx

# Uninstall but keep history
helm uninstall my-nginx --keep-history

# Uninstall from specific namespace
helm uninstall my-nginx -n web
```

### Working with Chart Values

**View default values:**
```bash
helm show values bitnami/nginx > default-values.yaml
```

**Custom values file (values.yaml):**
```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: true
  hostname: nginx.example.com
```

**Install with custom values:**
```bash
helm install my-nginx bitnami/nginx -f values.yaml
```

---

## 4. Use Kustomize for Configuration Management

Kustomize is a template-free way to customize Kubernetes objects. It's built into kubectl.

### Kustomize Concepts

- **Base**: Original/common configuration
- **Overlay**: Environment-specific customizations
- **Kustomization.yaml**: Configuration file defining resources and transformations

### Basic Kustomize Structure

```
myapp/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── development/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        ├── kustomization.yaml
        └── increase-replicas.yaml
```

### Base Configuration

**base/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app: myapp
```

**base/deployment.yaml:**
```yaml
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
        image: myapp:v1
        ports:
        - containerPort: 8080
```

**base/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: myapp
```

### Overlay for Development

**overlays/development/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: dev-
nameSuffix: -v1

commonLabels:
  environment: development

images:
  - name: myapp
    newTag: dev-latest
```

### Overlay for Production

**overlays/production/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

commonLabels:
  environment: production

replicas:
  - name: myapp
    count: 5

images:
  - name: myapp
    newTag: v2.0.0

patchesStrategicMerge:
  - increase-replicas.yaml
```

**overlays/production/increase-replicas.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
```

### Kustomize Commands

```bash
# Preview rendered manifests
kubectl kustomize overlays/production/

# Apply kustomization
kubectl apply -k overlays/production/

# Delete resources created by kustomization
kubectl delete -k overlays/production/

# Build and save to file
kubectl kustomize overlays/production/ > production-manifests.yaml
```

### Common Kustomize Transformations

**Add labels to all resources:**
```yaml
commonLabels:
  app.kubernetes.io/name: myapp
  app.kubernetes.io/version: "1.0"
```

**Add annotations:**
```yaml
commonAnnotations:
  description: "My application"
```

**Change image tags:**
```yaml
images:
  - name: myapp
    newName: registry.example.com/myapp
    newTag: v2.0.0
  - name: nginx
    newTag: "1.25"
```

**Add name prefix/suffix:**
```yaml
namePrefix: staging-
nameSuffix: -v2
```

**Set namespace:**
```yaml
namespace: production
```

**Add ConfigMaps:**
```yaml
configMapGenerator:
  - name: app-config
    literals:
      - DATABASE_URL=postgres://localhost:5432/db
    files:
      - config.properties
```

**Add Secrets:**
```yaml
secretGenerator:
  - name: app-secret
    literals:
      - password=secret123
    files:
      - credentials.txt
```

### Strategic Merge Patches

**patchesStrategicMerge** merges patches with base:

```yaml
# kustomization.yaml
patchesStrategicMerge:
  - add-sidecar.yaml

# add-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: logging-sidecar
        image: fluentd:v1
```

### JSON Patches

**patchesJson6902** for precise modifications:

```yaml
# kustomization.yaml
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: NEW_VAR
          value: "new-value"
```

---

## 5. Essential kubectl Commands for This Domain

```bash
# Deployment Management
kubectl create deployment nginx --image=nginx --replicas=3
kubectl set image deployment/nginx nginx=nginx:1.25
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx

# Scaling
kubectl scale deployment/nginx --replicas=5
kubectl autoscale deployment/nginx --min=2 --max=10 --cpu-percent=80

# Labels and Selectors
kubectl get pods -l app=myapp
kubectl get pods -l 'environment in (production, staging)'
kubectl label pods mypod tier=frontend
kubectl patch service myapp -p '{"spec":{"selector":{"version":"v2"}}}'

# Helm Commands
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
helm install my-release bitnami/nginx
helm upgrade my-release bitnami/nginx --set replicaCount=3
helm rollback my-release 1
helm uninstall my-release
helm list -A
helm history my-release

# Kustomize Commands
kubectl kustomize overlays/production/
kubectl apply -k overlays/production/
kubectl delete -k overlays/production/

# Service Management
kubectl expose deployment nginx --port=80 --target-port=8080
kubectl get endpoints
kubectl describe service nginx
```

---

## Exam Tips for This Domain

1. **Know rolling update parameters**: maxSurge and maxUnavailable control update behavior
2. **Practice rollbacks**: `kubectl rollout undo` is essential for quick recovery
3. **Understand blue/green vs canary**: Blue/green = instant switch, canary = gradual migration
4. **Master basic Helm commands**: install, upgrade, rollback, list, uninstall
5. **Know Helm value precedence**: --set > -f values.yaml > chart defaults
6. **Practice Kustomize overlays**: Base + overlays is a common exam pattern
7. **Use `kubectl apply -k`**: Built-in Kustomize support in kubectl
8. **Remember label selectors**: They control traffic routing in blue/green and canary

---

## Practice Exercises

1. Create a Deployment with 3 replicas, perform a rolling update to a new image version, then rollback to the previous version
2. Implement a blue/green deployment: create blue deployment with v1, deploy green with v2, switch traffic
3. Implement a canary deployment: start with 10% traffic to new version, gradually increase to 100%
4. Use Helm to install nginx from bitnami, customize the service type to NodePort, then upgrade with increased replicas
5. Create a Kustomize base with a Deployment and Service, then create overlays for dev (1 replica) and prod (5 replicas)
6. Pause a Deployment, make multiple changes (image, resources, env vars), then resume
7. Use `kubectl rollout history` to view revisions and rollback to a specific revision
8. Create a Helm release with custom values from a file, then upgrade with additional --set overrides