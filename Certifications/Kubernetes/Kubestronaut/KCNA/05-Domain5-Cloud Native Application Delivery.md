# KCNA Exam Study Guide: Domain 5 - Cloud Native Application Delivery

## Domain Weight: 8% (~5 questions out of 60)

This domain covers CI/CD practices, GitOps, deployment strategies, and tools used for delivering applications in cloud native environments.

---

## 5.1 Application Delivery Fundamentals

### What is Application Delivery?
The process of getting applications from development to production, including:
- Building and packaging applications
- Testing and validation
- Deploying to environments
- Managing releases and rollbacks

### Traditional vs Cloud Native Delivery

| Aspect | Traditional | Cloud Native |
|--------|-------------|--------------|
| **Frequency** | Monthly/Quarterly | Daily/Hourly |
| **Process** | Manual | Automated |
| **Rollback** | Difficult | Easy |
| **Testing** | End of cycle | Continuous |
| **Infrastructure** | Static | Dynamic |
| **Artifacts** | JARs, WAR files | Container images |

---

## 5.2 Continuous Integration (CI)

### What is CI?
Practice of frequently integrating code changes into a shared repository, with automated builds and tests.

### CI Pipeline Stages

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Code   │──►│  Build  │──►│  Test   │──►│  Scan   │──►│ Publish │
│  Commit │   │         │   │         │   │         │   │ Artifact│
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
```

### CI Key Practices
1. **Commit frequently**: Small, incremental changes
2. **Automated builds**: Triggered on every commit
3. **Automated testing**: Unit, integration, and e2e tests
4. **Fast feedback**: Fail fast, notify developers
5. **Maintain passing builds**: Never leave the build broken

### CI for Containers

**Dockerfile Build**:
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm test
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

**CI Pipeline Steps for Containers**:
1. Build container image
2. Run tests inside container
3. Scan image for vulnerabilities
4. Push to container registry
5. Sign image (optional)

### Popular CI Tools

| Tool | Description |
|------|-------------|
| **Jenkins** | Open-source automation server |
| **GitHub Actions** | CI/CD integrated with GitHub |
| **GitLab CI** | CI/CD integrated with GitLab |
| **CircleCI** | Cloud-based CI/CD platform |
| **Tekton** | Kubernetes-native CI/CD (CNCF) |
| **Azure DevOps** | Microsoft's CI/CD platform |

---

## 5.3 Continuous Delivery (CD)

### What is Continuous Delivery?
Automated process that ensures code can be released to production at any time.

### Continuous Delivery vs Continuous Deployment

| Aspect | Continuous Delivery | Continuous Deployment |
|--------|--------------------|-----------------------|
| **Production Deploy** | Manual approval | Fully automated |
| **When** | Ready anytime | Every passing change |
| **Risk** | Lower (human gate) | Higher (fully automated) |
| **Speed** | Slightly slower | Fastest |

```
┌──────────────────────────────────────────────────────────────────┐
│                      CI Pipeline                                 │
│   Commit → Build → Test → Scan → Publish                         │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                   CD Pipeline                                    │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ Continuous Delivery                                      │   │
│   │   Deploy to Staging → Testing → [Manual Approval] → Prod │   │
│   └──────────────────────────────────────────────────────────┘   │
│                              OR                                  │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ Continuous Deployment                                    │   │
│   │   Deploy to Staging → Testing → Prod (Automatic)         │   │
│   └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5.4 GitOps

### What is GitOps?
An operational framework that uses Git as the single source of truth for declarative infrastructure and applications.

### Core GitOps Principles

1. **Declarative**: Entire system described declaratively
2. **Versioned and Immutable**: Desired state stored in Git
3. **Pulled Automatically**: Software agents pull desired state
4. **Continuously Reconciled**: Agents ensure actual state matches desired state

### GitOps Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                         Developer                               │
│                            │                                    │
│                            ▼                                    │
│                    ┌───────────────┐                            │
│                    │  Git Commit   │                            │
│                    │ (manifests)   │                            │
│                    └───────┬───────┘                            │
│                            │                                    │
│                            ▼                                    │
│                    ┌───────────────┐                            │
│                    │   Git Repo    │ ◄── Source of Truth        │
│                    │ (GitOps Repo) │                            │
│                    └───────┬───────┘                            │
│                            │                                    │
│                            ▼ Pull                               │
│                    ┌───────────────┐                            │
│                    │  GitOps Agent │                            │
│                    │ (Argo/Flux)   │                            │
│                    └───────┬───────┘                            │
│                            │ Apply                              │
│                            ▼                                    │
│                    ┌───────────────┐                            │
│                    │  Kubernetes   │                            │
│                    │   Cluster     │                            │
│                    └───────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### Benefits of GitOps
- ✅ **Audit trail**: All changes tracked in Git
- ✅ **Rollback**: Easy revert to previous state
- ✅ **Consistency**: Same process for all environments
- ✅ **Security**: No direct cluster access needed
- ✅ **Collaboration**: Standard Git workflows (PRs, reviews)

### GitOps Tools

#### Argo CD (CNCF Graduated)
- **Purpose**: Declarative GitOps CD for Kubernetes
- **Features**:
  - Web UI for visualization
  - Multi-cluster support
  - SSO integration
  - Automated sync and self-healing
  - Health status monitoring

**Argo CD Application Example**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### Flux CD (CNCF Graduated)
- **Purpose**: GitOps toolkit for Kubernetes
- **Features**:
  - Kubernetes controller-based
  - Image automation
  - Helm chart support
  - Multi-tenancy

**Flux GitRepository Example**:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/repo.git
  ref:
    branch: main
```

### Argo CD vs Flux CD

| Feature | Argo CD | Flux CD |
|---------|---------|---------|
| **UI** | Built-in web UI | Separate (Weave GitOps) |
| **Architecture** | Application-centric | Toolkit-based |
| **Learning Curve** | Easier for beginners | More flexible |
| **Multi-Cluster** | Built-in | Via Kustomize |
| **Helm Support** | Yes | Yes |
| **CNCF Status** | Graduated | Graduated |

---

## 5.5 Deployment Strategies

### Rolling Update (Default in Kubernetes)

**How it works**: Gradually replaces old Pods with new ones

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Max pods above desired
      maxUnavailable: 25%  # Max pods unavailable
```

**Pros**: Zero downtime, easy rollback
**Cons**: Slow for large deployments, multiple versions running simultaneously

### Recreate Strategy

**How it works**: Terminates all old Pods before creating new ones

```yaml
spec:
  strategy:
    type: Recreate
```

**Pros**: Simple, no version mixing
**Cons**: Downtime during update

### Blue-Green Deployment

**How it works**: Run two identical environments, switch traffic instantly

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   ┌─────────────┐                                   │
│   │   Blue      │ ◄── Current (Production)          │
│   │ (v1.0.0)    │                                   │
│   └─────────────┘                                   │
│                                                     │
│   ┌─────────────┐                                   │
│   │   Green     │ ◄── New Version (Staging)         │
│   │ (v1.1.0)    │                                   │
│   └─────────────┘                                   │
│                                                     │
│   [Switch Traffic: Blue → Green]                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Pros**: Instant rollback, full testing of new version
**Cons**: Requires double resources

### Canary Deployment

**How it works**: Gradually shift traffic to new version

```
Stage 1:  [▓▓▓▓▓▓▓▓▓░]  10% new, 90% old
Stage 2:  [▓▓▓▓▓▓▓░░░]  30% new, 70% old
Stage 3:  [▓▓▓▓▓░░░░░]  50% new, 50% old
Stage 4:  [▓▓▓░░░░░░░]  70% new, 30% old
Final:    [░░░░░░░░░░]  100% new
```

**Pros**: Minimize risk, detect issues early
**Cons**: Complex traffic management, slower rollout

### A/B Testing

**How it works**: Route specific users to different versions

**Use Cases**:
- Feature testing
- UI/UX experiments
- Performance comparison

**Difference from Canary**:
- Canary: Random traffic split
- A/B: Targeted routing (by user, location, etc.)

---

## 5.6 Helm - Kubernetes Package Manager

### What is Helm?
- **Status**: CNCF Graduated
- **Purpose**: Package manager for Kubernetes
- **Analogy**: Like apt/yum for Kubernetes

### Helm Concepts

| Concept | Description |
|---------|-------------|
| **Chart** | Package containing K8s resources |
| **Release** | Instance of a chart in a cluster |
| **Repository** | Collection of charts |
| **Values** | Configuration for a chart |

### Helm Chart Structure
```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── charts/             # Chart dependencies
├── templates/          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── README.md
```

### Common Helm Commands
```bash
# Search for charts
helm search repo nginx
helm search hub wordpress

# Install a chart
helm install my-release bitnami/nginx

# Install with custom values
helm install my-release bitnami/nginx -f values.yaml

# List releases
helm list

# Upgrade a release
helm upgrade my-release bitnami/nginx --set replicaCount=3

# Rollback
helm rollback my-release 1

# Uninstall
helm uninstall my-release

# Create new chart
helm create mychart

# Package a chart
helm package mychart

# Lint a chart
helm lint mychart
```

### Values and Templating

**values.yaml**:
```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```

**Template Example** (deployment.yaml):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: nginx
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

---

## 5.7 Kustomize

### What is Kustomize?
- **Built into kubectl**: No separate tool needed
- **Purpose**: Template-free customization of Kubernetes manifests
- **Approach**: Patching and overlaying base configurations

### Kustomize vs Helm

| Aspect | Kustomize | Helm |
|--------|-----------|------|
| **Approach** | Overlay/Patch | Templates |
| **Learning Curve** | Lower | Higher |
| **Packaging** | No packaging | Charts |
| **Dependencies** | Built into kubectl | Separate tool |
| **Use Case** | Environment variations | Reusable packages |

### Kustomize Structure
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
        └── kustomization.yaml
```

### Base kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app: myapp
```

### Overlay kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

replicas:
  - name: myapp
    count: 5

images:
  - name: myapp
    newTag: v2.0.0
```

### Kustomize Commands
```bash
# Preview output
kubectl kustomize overlays/production

# Apply directly
kubectl apply -k overlays/production

# Build with kustomize
kustomize build overlays/production | kubectl apply -f -
```

---

## 5.8 Container Registries

### What is a Container Registry?
Storage and distribution system for container images.

### Public Registries

| Registry | URL | Description |
|----------|-----|-------------|
| **Docker Hub** | hub.docker.com | Default, largest public registry |
| **Quay.io** | quay.io | Red Hat's registry |
| **GitHub Container Registry** | ghcr.io | GitHub's registry |
| **Google Container Registry** | gcr.io | Google Cloud registry |

### Private/Managed Registries

| Registry | Provider |
|----------|----------|
| **Amazon ECR** | AWS |
| **Azure Container Registry** | Azure |
| **Google Artifact Registry** | GCP |
| **Harbor** | Open Source (CNCF) |
| **JFrog Artifactory** | JFrog |

### Image Naming Convention
```
[registry/][repository/]image:tag
```

Examples:
- `nginx:latest` (Docker Hub default)
- `gcr.io/project/myapp:v1.2.3`
- `myregistry.azurecr.io/app:abc123`

### Image Security
- **Vulnerability Scanning**: Trivy, Clair, Snyk
- **Image Signing**: Cosign, Notary
- **SBOM**: Software Bill of Materials

---

## 5.9 CI/CD Pipeline Example

### Complete Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                         CI PHASE                                 │
│                                                                  │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐           │
│  │ Commit  │──►│  Build  │──►│  Test   │──►│  Scan   │           │
│  │         │   │ Image   │   │ (Unit)  │   │ (Trivy) │           │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘           │
│                                                │                 │
│                                                ▼                 │
│                                        ┌─────────────┐           │
│                                        │    Push     │           │
│                                        │  to Registry│           │
│                                        └──────┬──────┘           │
└───────────────────────────────────────────────┼──────────────────┘
                                                │
                                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                         CD PHASE                                 │
│                                                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────────┐ │
│  │ Update Git  │──►│ GitOps Sync │──►│ Deploy to Kubernetes    │ │
│  │ (manifests) │   │ (Argo/Flux) │   │ (Rolling/Canary/B-G)    │ │
│  └─────────────┘   └─────────────┘   └─────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### GitHub Actions Example
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Run tests
        run: docker run myapp:${{ github.sha }} npm test
      
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
      
      - name: Push to registry
        run: |
          docker tag myapp:${{ github.sha }} ghcr.io/org/myapp:${{ github.sha }}
          docker push ghcr.io/org/myapp:${{ github.sha }}
      
      - name: Update manifests
        run: |
          # Update image tag in GitOps repo
          yq e '.spec.template.spec.containers[0].image = "ghcr.io/org/myapp:${{ github.sha }}"' \
            -i manifests/deployment.yaml
          git commit -am "Update to ${{ github.sha }}"
          git push
```

---

## Key Exam Tips for This Domain

1. **Understand CI vs CD** and their differences
2. **Know GitOps principles** and why Git is the source of truth
3. **Know Argo CD and Flux CD** as GitOps tools
4. **Understand deployment strategies**: Rolling, Blue-Green, Canary
5. **Know Helm basics**: Charts, releases, values
6. **Understand Kustomize** and how it differs from Helm
7. **Know the CI/CD pipeline stages** for containers

---

## Practice Questions

1. What is the difference between Continuous Delivery and Continuous Deployment?
   - Answer: Continuous Delivery requires manual approval for production, Continuous Deployment is fully automated

2. What is the source of truth in GitOps?
   - Answer: Git repository

3. Name two CNCF graduated GitOps tools.
   - Answer: Argo CD and Flux CD

4. What deployment strategy gradually shifts traffic to a new version?
   - Answer: Canary deployment

5. What is a Helm Chart?
   - Answer: A package containing Kubernetes resource definitions

6. What command applies Kustomize overlays?
   - Answer: `kubectl apply -k <path>` or `kustomize build <path>`

7. What is the difference between Blue-Green and Canary deployments?
   - Answer: Blue-Green switches all traffic at once, Canary gradually shifts traffic

8. Which tool is built into kubectl for customizing manifests?
   - Answer: Kustomize

---

## Quick Reference: Application Delivery Tools

| Category | Tool | Purpose |
|----------|------|---------|
| **CI** | Jenkins, GitHub Actions | Build and test |
| **CD/GitOps** | Argo CD, Flux CD | Deploy to Kubernetes |
| **Packaging** | Helm | Kubernetes package manager |
| **Customization** | Kustomize | Manifest overlays |
| **Registry** | Harbor, ECR, GCR | Store container images |
| **Pipelines** | Tekton | Kubernetes-native CI/CD |

---

## Additional Resources

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Flux CD Documentation](https://fluxcd.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [GitOps Principles](https://www.gitops.tech/)
- [Continuous Delivery Foundation](https://cd.foundation/)