# KCSA Exam Study Guide: Domain 1 - Overview of Cloud Native Security

## Domain Weight: 14% (~8 questions out of 60)

This domain covers the foundational concepts of cloud native security, including the 4Cs model, cloud provider security, isolation techniques, and workload protection.

---

## 1.1 The 4Cs of Cloud Native Security

### Security Layers Model

The 4Cs represent the four layers of cloud native security, from the outermost to innermost:

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLOUD                                   │
│  (Infrastructure: AWS, GCP, Azure, On-Premises)                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      CLUSTER                            │    │
│  │  (Kubernetes: Control Plane, Worker Nodes)              │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │                  CONTAINER                      │    │    │
│  │  │  (Images, Runtimes, Registries)                 │    │    │
│  │  │  ┌─────────────────────────────────────────┐    │    │    │
│  │  │  │                 CODE                    │    │    │    │
│  │  │  │  (Application, Dependencies)            │    │    │    │
│  │  │  └─────────────────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 4Cs Detailed Breakdown

| Layer | Description | Key Security Concerns |
|-------|-------------|----------------------|
| **Cloud** | Infrastructure layer | IAM, network security, encryption, compliance |
| **Cluster** | Kubernetes layer | API server security, RBAC, network policies |
| **Container** | Runtime layer | Image security, runtime security, isolation |
| **Code** | Application layer | Secure coding, dependency scanning, secrets |

### Cloud Layer Security
- **Infrastructure Security**: Physical security, network isolation
- **IAM (Identity and Access Management)**: Control who can access cloud resources
- **Encryption**: Data at rest and in transit
- **Network Security**: VPCs, firewalls, security groups
- **Compliance**: SOC2, ISO 27001, FedRAMP

### Cluster Layer Security
- **API Server Security**: Authentication, authorization, admission control
- **RBAC**: Role-based access control
- **Network Policies**: Pod-to-pod traffic control
- **Secrets Management**: Encrypted secrets, external vaults
- **Audit Logging**: Track all cluster activities

### Container Layer Security
- **Image Security**: Vulnerability scanning, signed images
- **Runtime Security**: Seccomp, AppArmor, SELinux
- **Registry Security**: Private registries, access control
- **Resource Limits**: Prevent resource exhaustion

### Code Layer Security
- **Secure Coding Practices**: OWASP guidelines
- **Dependency Scanning**: SCA (Software Composition Analysis)
- **Static Analysis**: SAST tools
- **Secrets in Code**: Never commit secrets

---

## 1.2 Cloud Provider and Infrastructure Security

### Shared Responsibility Model

```
┌───────────────────────────────────────────────────────────────┐
│                    CUSTOMER RESPONSIBILITY                    │
│  • Application security                                       │
│  • Data encryption                                            │
│  • Identity & access management                               │
│  • Operating system, network, firewall configuration          │
│  • Client-side data encryption                                │
├───────────────────────────────────────────────────────────────┤
│                CLOUD PROVIDER RESPONSIBILITY                  │
│  • Physical security of data centers                          │
│  • Hardware & infrastructure                                  │
│  • Network infrastructure                                     │
│  • Hypervisor security                                        │
│  • Global infrastructure                                      │
└───────────────────────────────────────────────────────────────┘
```

### Cloud Security Best Practices

**Identity and Access Management (IAM)**:
- Implement least privilege principle
- Use MFA (Multi-Factor Authentication)
- Rotate credentials regularly
- Use service accounts for workloads
- Audit access regularly

**Network Security**:
- Use VPCs (Virtual Private Clouds)
- Implement network segmentation
- Configure security groups/firewalls
- Use private endpoints
- Restrict public access

**Data Protection**:
- Encrypt data at rest
- Encrypt data in transit (TLS)
- Use key management services
- Implement backup and recovery

### Kubernetes on Cloud Providers

| Provider | Managed K8s Service | Security Features |
|----------|--------------------|--------------------|
| AWS | Amazon EKS | IAM integration, VPC, KMS |
| GCP | Google GKE | Workload Identity, VPC-native |
| Azure | Azure AKS | Azure AD, Private clusters |

---

## 1.3 Controls and Frameworks

### Security Control Types

| Control Type | Description | Examples |
|--------------|-------------|----------|
| **Preventive** | Stop attacks before they happen | Network policies, RBAC |
| **Detective** | Identify attacks in progress | Audit logs, IDS |
| **Corrective** | Respond to and fix issues | Incident response, patching |
| **Deterrent** | Discourage attacks | Security warnings, policies |

### Key Security Frameworks for Cloud Native

**OWASP Kubernetes Top 10**:
1. Insecure Workload Configurations
2. Supply Chain Vulnerabilities
3. Overly Permissive RBAC Configurations
4. Lack of Centralized Policy Enforcement
5. Inadequate Logging and Monitoring
6. Broken Authentication Mechanisms
7. Missing Network Segmentation Controls
8. Secrets Management Failures
9. Misconfigured Cluster Components
10. Outdated and Vulnerable Kubernetes Components

**CIS Kubernetes Benchmark**:
- Industry-standard security configuration guidelines
- Covers control plane, worker nodes, policies
- Automated scanning with kube-bench

**NIST Cybersecurity Framework**:
- Identify, Protect, Detect, Respond, Recover
- SP 800-190: Container Security Guide

---

## 1.4 Isolation Techniques

### Namespace Isolation

**Purpose**: Logical separation of resources within a cluster

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development
```

**Best Practices**:
- Separate environments (dev, staging, prod)
- Separate teams/tenants
- Apply resource quotas per namespace
- Use Network Policies per namespace

### Network Isolation

**Network Policies**: Control pod-to-pod communication

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Runtime Isolation

| Technique | Description | Use Case |
|-----------|-------------|----------|
| **Namespaces** | Logical isolation | Multi-tenancy |
| **Network Policies** | Network isolation | Zero-trust networking |
| **Pod Security** | Restrict pod capabilities | Security hardening |
| **Sandboxed Runtimes** | Strong isolation | Untrusted workloads |
| **Node Isolation** | Dedicated nodes | Sensitive workloads |

### Sandboxed Container Runtimes

- **gVisor**: User-space kernel for containers
- **Kata Containers**: VM-based isolation
- **Firecracker**: Lightweight VM-based isolation

---

## 1.5 Artifact Repository and Image Security

### Container Image Security

**Image Vulnerabilities**:
- Base image vulnerabilities
- Application dependencies
- Malicious packages
- Outdated software

**Best Practices**:
1. Use minimal base images (distroless, Alpine)
2. Scan images for vulnerabilities
3. Sign and verify images
4. Use private registries
5. Implement image policies

### Image Scanning Tools

| Tool | Type | Description |
|------|------|-------------|
| **Trivy** | Open Source | Comprehensive vulnerability scanner |
| **Clair** | Open Source | Container vulnerability analysis |
| **Grype** | Open Source | Vulnerability scanner for images |
| **Snyk** | Commercial | Developer-first security |
| **Aqua Security** | Commercial | Full container security |

### Image Signing and Verification

**Cosign** (Sigstore):
```bash
# Sign an image
cosign sign --key cosign.key myregistry/myimage:tag

# Verify an image
cosign verify --key cosign.pub myregistry/myimage:tag
```

**Image Pinning**:
```yaml
# Use digest instead of tag
image: nginx@sha256:abc123...
```

### Private Registry Security

- Enable authentication
- Use TLS/HTTPS
- Implement access control
- Enable vulnerability scanning
- Audit registry access

**Popular Private Registries**:
- Harbor (CNCF)
- Amazon ECR
- Google Container Registry / Artifact Registry
- Azure Container Registry
- JFrog Artifactory

---

## 1.6 Workload and Application Code Security

### Secure Development Practices

**SDLC Security Integration**:
```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Plan   │──►│  Code   │──►│  Build  │──►│  Test   │──►│ Deploy  │
│         │   │         │   │         │   │         │   │         │
│Security │   │SAST     │   │SCA      │   │DAST     │   │Runtime  │
│Require- │   │Code     │   │Depend-  │   │Pen      │   │Security │
│ments    │   │Review   │   │ency     │   │Testing  │   │Monitor  │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
```

### Application Security Testing

| Type | Name | When | What |
|------|------|------|------|
| **SAST** | Static Application Security Testing | Build time | Code analysis |
| **DAST** | Dynamic Application Security Testing | Runtime | Running app testing |
| **SCA** | Software Composition Analysis | Build time | Dependency analysis |
| **IAST** | Interactive Application Security Testing | Runtime | Combined approach |

### Dependency Security

- Keep dependencies updated
- Use dependency scanning (npm audit, pip-audit)
- Pin dependency versions
- Use lock files
- Monitor for CVEs

### Security Context in Kubernetes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

---

## 1.7 Cloud Native Security Tools

### CNCF Security Projects

| Project | Category | Status | Purpose |
|---------|----------|--------|---------|
| **Falco** | Runtime Security | Graduated | Threat detection |
| **OPA** | Policy | Graduated | Policy enforcement |
| **Notary/TUF** | Signing | Graduated | Content trust |
| **cert-manager** | Certificates | Incubating | Certificate management |
| **Kyverno** | Policy | Incubating | K8s-native policy |
| **in-toto** | Supply Chain | Incubating | Supply chain integrity |
| **SPIFFE/SPIRE** | Identity | Graduated | Workload identity |

### Security Tool Categories

**Vulnerability Scanning**:
- Trivy, Clair, Grype, Snyk

**Runtime Security**:
- Falco, Sysdig, Aqua

**Policy Enforcement**:
- OPA/Gatekeeper, Kyverno

**Compliance Scanning**:
- kube-bench, Kubescape, Polaris

**Secrets Management**:
- HashiCorp Vault, AWS Secrets Manager, External Secrets Operator

---

## Key Exam Tips for This Domain

1. **Memorize the 4Cs**: Cloud, Cluster, Container, Code (outer to inner)
2. **Understand shared responsibility** between cloud provider and customer
3. **Know isolation techniques**: Namespaces, Network Policies, Pod Security
4. **Understand image security**: Scanning, signing, private registries
5. **Know the OWASP Kubernetes Top 10** categories
6. **Understand the difference** between SAST, DAST, and SCA

---

## Practice Questions

1. What are the 4Cs of Cloud Native Security (from outer to inner)?
   - Answer: Cloud, Cluster, Container, Code

2. Which isolation technique controls pod-to-pod network traffic?
   - Answer: Network Policies

3. What type of testing analyzes source code without executing it?
   - Answer: SAST (Static Application Security Testing)

4. Which CNCF graduated project is used for runtime threat detection?
   - Answer: Falco

5. What is the purpose of image signing?
   - Answer: To verify the source and integrity of container images

6. Which tool is used to scan clusters against CIS benchmarks?
   - Answer: kube-bench

7. What is the shared responsibility model?
   - Answer: Division of security responsibilities between cloud provider and customer

8. Name two sandboxed container runtimes.
   - Answer: gVisor, Kata Containers

---

## Additional Resources

- [Kubernetes Cloud Native Security](https://kubernetes.io/docs/concepts/security/cloud-native-security/)
- [OWASP Kubernetes Top 10](https://owasp.org/www-project-kubernetes-top-ten/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NIST Container Security Guide (SP 800-190)](https://csrc.nist.gov/publications/detail/sp/800-190/final)