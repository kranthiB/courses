# CKS Domain 5: Supply Chain Security (20%)

## Overview

This domain focuses on container-oriented security throughout the software supply chain. It covers minimizing base image footprint, securing container registries, image scanning for vulnerabilities, static analysis, signing and validating images, and understanding SBOM (Software Bill of Materials).

---

## 5.1 Minimize Base Image Footprint

### Image Optimization Strategies

1. **Use minimal base images** - Alpine, distroless, scratch
2. **Multi-stage builds** - Separate build and runtime stages
3. **Remove unnecessary packages** - Only include what's needed
4. **Don't include secrets** - Never bake credentials into images

### Minimal Base Images Comparison

| Image | Size | Description |
|-------|------|-------------|
| `scratch` | 0 MB | Empty image, for statically compiled binaries |
| `alpine` | ~5 MB | Minimal Linux with musl libc and busybox |
| `distroless` | ~2-50 MB | Google's minimal images without package managers |
| `debian-slim` | ~75 MB | Stripped down Debian |
| `ubuntu` | ~75 MB | Full Ubuntu image |

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Runtime stage
FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .

# Create non-root user
RUN adduser -D -g '' appuser
USER appuser

EXPOSE 8080
CMD ["./main"]
```

### Distroless Image Example

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# Runtime stage - distroless
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

### Dockerfile Best Practices

```dockerfile
# 1. Use specific tags, not 'latest'
FROM alpine:3.19

# 2. Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 3. Remove unnecessary packages and caches
RUN apk add --no-cache python3 py3-pip && \
    pip install --no-cache-dir flask && \
    rm -rf /var/cache/apk/*

# 4. Copy only necessary files
COPY --chown=appuser:appgroup app.py /app/

# 5. Use minimal permissions
WORKDIR /app
USER appuser

# 6. Don't expose unnecessary ports
EXPOSE 8080

# 7. Use COPY instead of ADD (unless extracting)
COPY requirements.txt .

# 8. Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y --no-install-recommends package && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Dockerfile Security Checklist

```dockerfile
# BAD - Security issues
FROM ubuntu:latest                    # Don't use 'latest'
RUN apt-get update && apt-get install -y curl vim ssh
ADD https://example.com/file.tar /app  # Don't use ADD for URLs
COPY ./secrets.env /app/               # Don't copy secrets
USER root                              # Don't run as root
EXPOSE 22 80 443 3306                  # Don't expose unnecessary ports

# GOOD - Secure Dockerfile
FROM alpine:3.19                       # Specific version
RUN apk add --no-cache curl           # Only needed packages
COPY app.py /app/                     # Use COPY, not ADD
RUN adduser -D appuser                # Create non-root user
USER appuser                          # Run as non-root
EXPOSE 8080                           # Only required port
```

---

## 5.2 Secure Your Supply Chain

### Understand Your Supply Chain (SBOM)

Software Bill of Materials (SBOM) provides visibility into software components:

```bash
# Generate SBOM using Trivy
trivy image --format spdx nginx:1.25
trivy image --format cyclonedx nginx:1.25

# Generate SBOM using Syft
syft nginx:1.25 -o spdx-json > nginx-sbom.json
syft nginx:1.25 -o cyclonedx-json > nginx-cyclonedx.json
```

### Whitelist Allowed Registries

#### Using Kyverno (Policy Engine)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: validate-registries
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Images must be from allowed registries"
      pattern:
        spec:
          containers:
          - image: "gcr.io/myproject/* | docker.io/myorg/* | registry.k8s.io/*"
          initContainers:
          - image: "gcr.io/myproject/* | docker.io/myorg/* | registry.k8s.io/*"
```

#### Using OPA/Gatekeeper

```yaml
# ConstraintTemplate
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("initContainer <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }
---
# Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - "production"
  parameters:
    repos:
    - "gcr.io/myproject/"
    - "docker.io/myorg/"
    - "registry.k8s.io/"
```

#### Using ImagePolicyWebhook

```yaml
# /etc/kubernetes/admission/imagepolicy.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/admission/imagepolicy-kubeconfig.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false  # Deny if webhook unavailable
```

```yaml
# /etc/kubernetes/admission/imagepolicy-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- name: image-checker
  cluster:
    certificate-authority: /etc/kubernetes/admission/ca.crt
    server: https://image-policy-webhook.kube-system.svc:443/check
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/admission/client.crt
    client-key: /etc/kubernetes/admission/client.key
contexts:
- context:
    cluster: image-checker
    user: api-server
  name: image-checker
current-context: image-checker
```

### Using Image Digests

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: digest-pod
spec:
  containers:
  - name: nginx
    # Use digest instead of tag for immutability
    image: nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d912703568daff790ab3f755
```

### Get Image Digest

```bash
# Get digest from running container
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].imageID}'

# Get digest from registry
docker pull nginx:1.25
docker inspect nginx:1.25 --format='{{index .RepoDigests 0}}'

# Using crane
crane digest nginx:1.25

# Using skopeo
skopeo inspect docker://docker.io/nginx:1.25 | jq -r '.Digest'
```

### Image Signing with Cosign

```bash
# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key myregistry.io/myimage:v1

# Verify signature
cosign verify --key cosign.pub myregistry.io/myimage:v1
```

### Enforce Signed Images with Kyverno

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "myregistry.io/myimage:*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
```

---

## 5.3 Scan Images for Known Vulnerabilities

### Trivy

Trivy is a comprehensive vulnerability scanner for containers and other artifacts.

#### Installation

```bash
# Install on Debian/Ubuntu
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Or download binary directly
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```

#### Basic Usage

```bash
# Scan an image
trivy image nginx:1.25

# Scan with severity filter
trivy image --severity HIGH,CRITICAL nginx:1.25

# Scan with exit code on findings
trivy image --exit-code 1 --severity CRITICAL nginx:1.25

# Output to JSON
trivy image --format json --output results.json nginx:1.25

# Scan a local Dockerfile
trivy config Dockerfile

# Scan filesystem
trivy fs /path/to/project

# Scan Kubernetes manifests
trivy config --policy /path/to/policies /path/to/k8s-manifests
```

#### Finding Specific CVEs

```bash
# Search for specific CVE
trivy image nginx:1.25 | grep -E "CVE-2023-44487|CVE-2023-38545"

# Filter by CVE ID
trivy image --vuln-type os nginx:1.25 2>/dev/null | grep CVE-2023

# Generate report with only specific severities
trivy image --severity CRITICAL nginx:1.25
```

#### Trivy Operator for Kubernetes

```bash
# Install Trivy Operator
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/trivy-operator/main/deploy/static/trivy-operator.yaml

# Check vulnerability reports
kubectl get vulnerabilityreports -A
kubectl describe vulnerabilityreport <report-name> -n <namespace>
```

### Scan for Vulnerabilities in CI/CD

```yaml
# Example GitHub Actions workflow
name: Container Security Scan
on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build image
      run: docker build -t myimage:${{ github.sha }} .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myimage:${{ github.sha }}'
        format: 'table'
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
```

---

## 5.4 Perform Static Analysis of User Workloads

### Kubesec

Kubesec performs security risk analysis for Kubernetes resources.

```bash
# Install
wget https://github.com/controlplaneio/kubesec/releases/download/v2.13.0/kubesec_linux_amd64.tar.gz
tar -xvf kubesec_linux_amd64.tar.gz
mv kubesec /usr/local/bin/

# Scan a manifest file
kubesec scan pod.yaml

# Scan from stdin
cat pod.yaml | kubesec scan -

# Using Docker
docker run -i kubesec/kubesec:latest scan /dev/stdin < pod.yaml

# Online API (for testing only)
curl -sSX POST --data-binary @"pod.yaml" https://v2.kubesec.io/scan
```

#### Understanding Kubesec Output

```json
{
  "object": "Pod/insecure-pod.default",
  "valid": true,
  "score": -30,
  "scoring": {
    "critical": [
      {
        "id": "CapSysAdmin",
        "selector": "containers[] .securityContext .capabilities .add == SYS_ADMIN",
        "reason": "CAP_SYS_ADMIN is the most privileged capability",
        "points": -30
      }
    ],
    "passed": [
      {
        "id": "ServiceAccountName",
        "selector": "spec .serviceAccountName",
        "reason": "Service account used",
        "points": 3
      }
    ],
    "advise": [
      {
        "id": "ApparmorAny",
        "selector": "metadata .annotations .container.apparmor.security.beta.kubernetes.io/nginx",
        "reason": "AppArmor not configured",
        "points": 3
      }
    ]
  }
}
```

### KubeLinter

KubeLinter analyzes Kubernetes YAML files and Helm charts for security misconfigurations.

```bash
# Install
curl -LO https://github.com/stackrox/kube-linter/releases/download/v0.6.4/kube-linter-linux.tar.gz
tar -xvf kube-linter-linux.tar.gz
mv kube-linter /usr/local/bin/

# Scan manifests
kube-linter lint /path/to/manifests/

# Scan with specific checks
kube-linter lint --include "run-as-non-root" /path/to/manifests/

# List available checks
kube-linter checks list

# Output as JSON
kube-linter lint --format json /path/to/manifests/
```

### Static Analysis Tools Comparison

| Tool | Purpose | Focus |
|------|---------|-------|
| Trivy | Vulnerability scanning | CVEs in images |
| Kubesec | Risk analysis | Security scoring |
| KubeLinter | Linting | Best practices |
| Checkov | IaC scanning | Misconfigurations |
| Conftest | Policy testing | OPA policies |

### Analyzing Dockerfiles

```bash
# Hadolint for Dockerfile linting
docker run --rm -i hadolint/hadolint < Dockerfile

# Dockle for image best practices
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock goodwithtech/dockle:latest myimage:tag
```

---

## 5.5 CI/CD Pipeline Security

### Secure Pipeline Practices

```yaml
# Example: Secure GitLab CI pipeline
stages:
  - build
  - test
  - scan
  - sign
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

vulnerability-scan:
  stage: scan
  script:
    - trivy image --exit-code 1 --severity CRITICAL $IMAGE_TAG
  allow_failure: false

static-analysis:
  stage: scan
  script:
    - kubesec scan k8s/deployment.yaml
    - kube-linter lint k8s/

sign-image:
  stage: sign
  script:
    - cosign sign --key $COSIGN_PRIVATE_KEY $IMAGE_TAG
  only:
    - main

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/
  only:
    - main
  when: manual
```

---

## Hands-On Lab Exercises

### Lab 1: Image Vulnerability Scanning with Trivy

```bash
# Scan multiple images and compare
trivy image nginx:1.25 --severity HIGH,CRITICAL
trivy image alpine:3.19 --severity HIGH,CRITICAL
trivy image ubuntu:22.04 --severity HIGH,CRITICAL

# Find images with specific CVEs
trivy image nginx:1.25 | grep -c CRITICAL
trivy image nginx:1.25 | grep -c HIGH

# Generate JSON report
trivy image --format json nginx:1.25 > nginx-scan.json

# Scan with exit code for CI/CD
trivy image --exit-code 1 --severity CRITICAL nginx:1.25 && echo "No critical vulnerabilities" || echo "Critical vulnerabilities found"
```

### Lab 2: Static Analysis with Kubesec

```bash
# Create an insecure pod manifest
cat > insecure-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
      runAsUser: 0
      capabilities:
        add:
        - SYS_ADMIN
EOF

# Scan with Kubesec
kubesec scan insecure-pod.yaml

# Create a secure pod manifest
cat > secure-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
EOF

# Compare scores
kubesec scan secure-pod.yaml
```

### Lab 3: Restrict Image Registries with Kyverno

```bash
# Install Kyverno
kubectl apply -f https://raw.githubusercontent.com/kyverno/kyverno/main/config/install.yaml

# Wait for Kyverno to be ready
kubectl wait --for=condition=ready pod -l app=kyverno -n kyverno --timeout=120s

# Create policy
kubectl apply -f - <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allowed-registries
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: validate-registry
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Unknown image registry"
      pattern:
        spec:
          containers:
          - image: "docker.io/* | gcr.io/* | registry.k8s.io/*"
EOF

# Test with allowed registry
kubectl run test-allowed --image=docker.io/nginx --dry-run=server

# Test with disallowed registry
kubectl run test-blocked --image=malicious.registry.io/nginx --dry-run=server
# Should be blocked
```

### Lab 4: Multi-Stage Docker Build

```bash
# Create a simple Go application
mkdir /tmp/go-app && cd /tmp/go-app

cat > main.go <<EOF
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, Secure World!")
    })
    http.ListenAndServe(":8080", nil)
}
EOF

cat > go.mod <<EOF
module myapp
go 1.21
EOF

# Create optimized Dockerfile
cat > Dockerfile <<EOF
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod .
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o server .

# Runtime stage
FROM alpine:3.19
RUN apk --no-cache add ca-certificates && \
    adduser -D -g '' appuser
WORKDIR /app
COPY --from=builder /app/server .
USER appuser
EXPOSE 8080
CMD ["./server"]
EOF

# Build image
docker build -t secure-go-app:v1 .

# Check image size
docker images secure-go-app:v1

# Scan the image
trivy image secure-go-app:v1
```

---

## Practice Questions

1. **Scan an image with Trivy** and filter results to show only CRITICAL vulnerabilities.

2. **Create a Kyverno policy** that ensures all pods use images from `gcr.io/my-project/` registry.

3. **Write a multi-stage Dockerfile** that builds a Node.js application with a minimal runtime image.

4. **Use kubesec** to analyze a pod manifest and explain the scoring.

5. **How do you verify** an image's digest before deployment?

---

## Quick Reference

### Trivy Commands

```bash
# Scan image
trivy image <image>

# Filter by severity
trivy image --severity HIGH,CRITICAL <image>

# Output format
trivy image --format json <image>

# Scan filesystem
trivy fs /path

# Scan config files
trivy config /path/to/k8s
```

### Kubesec Scoring

| Score | Risk Level |
|-------|------------|
| > 0 | Good |
| 0 | Neutral |
| < 0 | Risky |
| << -10 | Critical |

### Image Digest Format

```
<registry>/<repository>@sha256:<64-hex-chars>
```

Example:
```
nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d912703568daff790ab3f755
```

### Dockerfile Security Keywords

```dockerfile
FROM <minimal-image>    # Use minimal base
USER <non-root>         # Run as non-root
RUN <single-command>    # Minimize layers
COPY <files>            # Don't use ADD
EXPOSE <needed-ports>   # Only required ports
```

---

## Official Documentation References

*These URLs are accessible during the CKS exam:*

- [Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

*External resources (for study, not accessible during exam):*

- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Kubesec.io](https://kubesec.io/)
- [Kyverno Documentation](https://kyverno.io/docs/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)