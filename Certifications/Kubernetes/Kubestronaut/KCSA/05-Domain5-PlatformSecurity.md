# KCSA Exam Study Guide: Domain 5 - Platform Security

## Domain Weight: 16% (~10 questions out of 60)

This domain covers supply chain security, image security, observability for security, service mesh, PKI, and admission control.

---

## 5.1 Supply Chain Security

### Software Supply Chain Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SECURE SOFTWARE SUPPLY CHAIN                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SOURCE          BUILD           PACKAGE         DEPLOY             │
│  ┌──────┐       ┌──────┐        ┌──────┐       ┌──────┐             │
│  │ Code │──────►│ CI   │───────►│Image │──────►│ K8s  │             │
│  │ Repo │       │ Build│        │ Reg  │       │      │             │
│  └──────┘       └──────┘        └──────┘       └──────┘             │
│     │              │               │              │                 │
│     ▼              ▼               ▼              ▼                 │
│  ✓ Code review  ✓ SBOM         ✓ Sign image   ✓ Verify signature    │
│  ✓ Signed      ✓ Scan deps    ✓ Scan vulns   ✓ Admission control    │
│    commits     ✓ Isolated     ✓ Private      ✓ Runtime security     │
│  ✓ Branch        build          registry                            │
│    protection                                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Supply Chain Security Controls

| Phase | Security Control | Tools |
|-------|------------------|-------|
| **Source** | Code review, signed commits | GitHub, GitLab |
| **Dependencies** | Dependency scanning, lock files | npm audit, pip-audit, Snyk |
| **Build** | Isolated builds, SBOM generation | Tekton, Syft |
| **Image** | Vulnerability scanning, signing | Trivy, Cosign, Notary |
| **Registry** | Access control, scanning | Harbor, ECR, GCR |
| **Deploy** | Admission control, policy | Kyverno, OPA/Gatekeeper |
| **Runtime** | Threat detection | Falco, Sysdig |

### SLSA Framework (Supply-chain Levels for Software Artifacts)

| Level | Requirements |
|-------|--------------|
| **SLSA 1** | Documentation of build process |
| **SLSA 2** | Tamper-resistant build service, provenance |
| **SLSA 3** | Hardened builds, auditable provenance |
| **SLSA 4** | Two-party review, hermetic builds |

### Software Bill of Materials (SBOM)

**What is SBOM?**
- Complete list of components in software
- Includes dependencies, versions, licenses
- Enables vulnerability tracking
- Required by some regulations

**SBOM Formats**:
- SPDX (Software Package Data Exchange)
- CycloneDX
- SWID (Software Identification)

**Generate SBOM with Syft**:
```bash
# Generate SBOM for container image
syft nginx:latest -o cyclonedx-json > sbom.json

# Generate SBOM for directory
syft dir:/path/to/project -o spdx-json > sbom.json
```

---

## 5.2 Image Repository Security

### Container Registry Security

```
┌─────────────────────────────────────────────────────────────────┐
│                   SECURE IMAGE REGISTRY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Security Features:                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐   │
│  │ Authentication  │  │ Vulnerability   │  │ Image Signing  │   │
│  │ & Authorization │  │ Scanning        │  │ & Verification │   │
│  └─────────────────┘  └─────────────────┘  └────────────────┘   │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐   │
│  │ Content Trust   │  │ Immutable Tags  │  │ Access Audit   │   │
│  │ (Notary)        │  │                 │  │ Logging        │   │
│  └─────────────────┘  └─────────────────┘  └────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Private Registry Options

| Registry | Type | Features |
|----------|------|----------|
| **Harbor** | CNCF | Scanning, signing, replication, RBAC |
| **Amazon ECR** | Cloud | IAM integration, scanning |
| **Google Artifact Registry** | Cloud | Multi-format, scanning |
| **Azure ACR** | Cloud | Azure AD, geo-replication |
| **JFrog Artifactory** | Commercial | Universal, security scanning |
| **Quay.io** | Red Hat | Security scanning, mirroring |

### Image Security Best Practices

**1. Use Minimal Base Images**:
```dockerfile
# ❌ Full OS image
FROM ubuntu:latest

# ✅ Minimal image
FROM gcr.io/distroless/static:nonroot

# ✅ Alpine (small)
FROM alpine:3.18
```

**2. Multi-stage Builds**:
```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp

# Runtime stage (minimal)
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/myapp /
USER nonroot:nonroot
ENTRYPOINT ["/myapp"]
```

**3. Pin Image Versions**:
```yaml
# ❌ Mutable tag
image: nginx:latest

# ✅ Specific version
image: nginx:1.25.3

# ✅ Digest (immutable)
image: nginx@sha256:abc123def456...
```

**4. Scan Images**:
```bash
# Trivy scan
trivy image nginx:latest

# Grype scan
grype nginx:latest
```

### Image Signing with Cosign

```bash
# Generate key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key myregistry/myimage:tag

# Verify an image
cosign verify --key cosign.pub myregistry/myimage:tag
```

### Pull Image from Private Registry

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg-pod
spec:
  containers:
  - name: app
    image: myregistry.io/myimage:tag
  imagePullSecrets:
  - name: regcred
---
# Create registry secret
# kubectl create secret docker-registry regcred \
#   --docker-server=myregistry.io \
#   --docker-username=user \
#   --docker-password=pass
```

---

## 5.3 Observability for Security

### Security Observability Pillars

```
┌─────────────────────────────────────────────────────────────────┐
│                   SECURITY OBSERVABILITY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LOGS                  METRICS               TRACES             │
│  ├─ Audit logs        ├─ Security events    ├─ Request tracing  │
│  ├─ Container logs    ├─ Failed auth        ├─ Attack paths     │
│  ├─ System logs       ├─ Policy violations  ├─ Incident         │
│  └─ Network logs      └─ Resource anomalies │   correlation     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    ALERTING & RESPONSE                      ││
│  │  Falco → Alert Manager → PagerDuty/Slack → Incident Response││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes Audit Logging

**Audit Policy Example**:
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all secret operations
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]

# Log all authentication failures
- level: RequestResponse
  users: ["system:anonymous"]
  verbs: ["*"]

# Log RBAC changes
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
```

### Security-Focused Observability Tools

| Tool | Purpose | Type |
|------|---------|------|
| **Falco** | Runtime threat detection | CNCF Graduated |
| **Prometheus** | Metrics collection | CNCF Graduated |
| **Fluentd/Fluent Bit** | Log collection | CNCF Graduated |
| **Jaeger** | Distributed tracing | CNCF Graduated |
| **OpenTelemetry** | Unified observability | CNCF Graduated |
| **Grafana** | Visualization | Open Source |
| **Loki** | Log aggregation | Open Source |

### Falco for Runtime Security

**Common Falco Rules**:
```yaml
# Detect privileged container
- rule: Launch Privileged Container
  desc: Detect privileged container launch
  condition: >
    spawned_process and container and
    container.privileged=true
  output: Privileged container started (user=%user.name container=%container.name)
  priority: WARNING

# Detect shell in container
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    spawned_process and container and
    shell_procs and proc.tty != 0
  output: Shell spawned in container (user=%user.name container=%container.name)
  priority: NOTICE

# Detect sensitive file access
- rule: Read sensitive file untrusted
  desc: Sensitive file opened by untrusted program
  condition: >
    sensitive_files and open_read and
    not proc.name in (trusted_readers)
  output: Sensitive file opened (file=%fd.name proc=%proc.name)
  priority: WARNING
```

### Security Metrics to Monitor

| Category | Metrics |
|----------|---------|
| **Authentication** | Failed logins, unusual login times |
| **Authorization** | RBAC denials, escalation attempts |
| **Network** | Unusual traffic patterns, policy violations |
| **Resources** | CPU/memory spikes (cryptomining) |
| **Pods** | Privileged pods, host mounts |
| **Images** | Vulnerability counts, unsigned images |

---

## 5.4 Service Mesh Security

### Service Mesh Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      SERVICE MESH                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONTROL PLANE (e.g., istiod)                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Configuration management                             │    │
│  │  • Certificate authority                                │    │
│  │  • Service discovery                                    │    │
│  │  • Policy distribution                                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                    │
│                            ▼                                    │
│  DATA PLANE (Sidecar proxies)                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ Pod A        │    │ Pod B        │    │ Pod C        │       │
│  │ ┌──────────┐ │    │ ┌──────────┐ │    │ ┌──────────┐ │       │
│  │ │ App      │ │    │ │ App      │ │    │ │ App      │ │       │
│  │ └──────────┘ │    │ └──────────┘ │    │ └──────────┘ │       │
│  │ ┌──────────┐ │    │ ┌──────────┐ │    │ ┌──────────┐ │       │
│  │ │ Envoy    │◄├────┼─┤ Envoy    │◄├────┼─┤ Envoy    │ │       │
│  │ │ Proxy    │ │mTLS│ │ Proxy    │ │mTLS│ │ Proxy    │ │       │
│  │ └──────────┘ │    │ └──────────┘ │    │ └──────────┘ │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Service Mesh Security Features

| Feature | Description |
|---------|-------------|
| **mTLS** | Mutual TLS between services (automatic encryption) |
| **Authentication** | Service-to-service identity verification |
| **Authorization** | Fine-grained access policies |
| **Traffic Encryption** | All traffic encrypted by default |
| **Certificate Management** | Automatic cert rotation |
| **Observability** | Detailed traffic metrics and tracing |

### mTLS (Mutual TLS)

**How mTLS Works**:
```
┌──────────┐                          ┌──────────┐
│ Service A│                          │ Service B│
│          │  1. Client Hello         │          │
│          │ ────────────────────────►│          │
│          │  2. Server Cert          │          │
│          │ ◄────────────────────────│          │
│          │  3. Client Cert          │          │
│          │ ────────────────────────►│          │
│          │  4. Encrypted Traffic    │          │
│          │ ◄───────────────────────►│          │
└──────────┘                          └──────────┘
```

**Benefits**:
- Both client and server authenticate
- All traffic encrypted
- Zero-trust networking
- No application code changes

### Istio Security Features

```yaml
# PeerAuthentication - Require mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# AuthorizationPolicy - Allow specific traffic
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: frontend
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/backend"]
```

### Service Mesh Comparison

| Feature | Istio | Linkerd |
|---------|-------|---------|
| **Proxy** | Envoy | linkerd-proxy (Rust) |
| **mTLS** | Yes | Yes |
| **Complexity** | Higher | Lower |
| **Resources** | More | Less |
| **L7 Policies** | Extensive | Basic |
| **CNCF Status** | Graduated | Graduated |

---

## 5.5 PKI (Public Key Infrastructure)

### Kubernetes PKI Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES PKI                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROOT CA                                                        │
│  └─► API Server Certificate                                     │
│  └─► API Server Client Certificate (for etcd)                   │
│  └─► Controller Manager Client Certificate                      │
│  └─► Scheduler Client Certificate                               │
│  └─► Kubelet Client Certificates                                │
│  └─► Service Account Signing Keys                               │
│                                                                 │
│  ETCD CA (Separate)                                             │
│  └─► etcd Server Certificate                                    │
│  └─► etcd Peer Certificates                                     │
│                                                                 │
│  FRONT PROXY CA (Optional)                                      │
│  └─► Front Proxy Client Certificate                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Certificate Management with cert-manager

**Install cert-manager**:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

**Create Issuer**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Request Certificate**:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: production
spec:
  secretName: myapp-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: myapp.example.com
  dnsNames:
  - myapp.example.com
  - www.myapp.example.com
```

---

## 5.6 Admission Control

### Admission Controller Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   ADMISSION CONTROL FLOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  API Request                                                    │
│      │                                                          │
│      ▼                                                          │
│  ┌──────────────────┐                                           │
│  │  Authentication  │                                           │
│  └──────────────────┘                                           │
│      │                                                          │
│      ▼                                                          │
│  ┌──────────────────┐                                           │
│  │  Authorization   │                                           │
│  └──────────────────┘                                           │
│      │                                                          │
│      ▼                                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              ADMISSION CONTROLLERS                       │   │
│  │  ┌────────────────────┐  ┌────────────────────────────┐  │   │
│  │  │ Mutating Webhooks  │─►│ Validating Webhooks        │  │   │
│  │  │ (Modify request)   │  │ (Accept/Reject request)    │  │   │
│  │  └────────────────────┘  └────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      ▼                                                          │
│  ┌──────────────────┐                                           │
│  │  Persist to etcd │                                           │
│  └──────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Built-in Admission Controllers

**Security-Related Controllers**:
| Controller | Purpose |
|------------|---------|
| **PodSecurity** | Enforce Pod Security Standards |
| **NodeRestriction** | Limit kubelet permissions |
| **LimitRanger** | Enforce resource limits |
| **ResourceQuota** | Enforce resource quotas |
| **AlwaysPullImages** | Force image pull |
| **ImagePolicyWebhook** | External image validation |
| **DenyServiceExternalIPs** | Block CVE-2020-8554 |

### Policy Engines

**OPA/Gatekeeper**:
```yaml
# Constraint Template
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing labels: %v", [missing])
      }

---
# Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    labels: ["team", "app"]
```

**Kyverno** (Kubernetes-native):
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-team-label
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Label 'team' is required"
      pattern:
        metadata:
          labels:
            team: "?*"
```

### OPA vs Kyverno

| Aspect | OPA/Gatekeeper | Kyverno |
|--------|----------------|---------|
| **Language** | Rego (custom) | YAML (native) |
| **Learning Curve** | Steeper | Easier |
| **Mutation** | Yes | Yes |
| **Validation** | Yes | Yes |
| **Generation** | Limited | Yes |
| **CNCF** | Graduated (OPA) | Incubating |

---

## Key Exam Tips for This Domain

1. **Supply chain security**: Know SLSA levels, SBOM, image signing
2. **Image security**: Scanning (Trivy), signing (Cosign), minimal images
3. **Service mesh**: mTLS provides encryption AND authentication
4. **Falco**: Runtime security, syscall monitoring
5. **Admission control**: Mutating → Validating order
6. **Policy engines**: OPA uses Rego, Kyverno uses YAML
7. **cert-manager**: Automates certificate lifecycle

---

## Practice Questions

1. What does SBOM stand for?
   - Answer: Software Bill of Materials

2. Which tool is used for signing container images?
   - Answer: Cosign (or Notary)

3. What does mTLS provide in a service mesh?
   - Answer: Mutual authentication and encryption

4. Which CNCF project is used for runtime threat detection?
   - Answer: Falco

5. What is the order of admission controller processing?
   - Answer: Mutating webhooks first, then Validating webhooks

6. Which policy engine uses the Rego language?
   - Answer: OPA (Open Policy Agent)

7. What CNCF project automates certificate management?
   - Answer: cert-manager

8. What type of base image has minimal attack surface?
   - Answer: Distroless images

---

## Additional Resources

- [Supply Chain Security - SLSA](https://slsa.dev/)
- [Sigstore/Cosign](https://docs.sigstore.dev/cosign/overview/)
- [Falco Documentation](https://falco.org/docs/)
- [Istio Security](https://istio.io/latest/docs/concepts/security/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
- [Kyverno](https://kyverno.io/docs/)