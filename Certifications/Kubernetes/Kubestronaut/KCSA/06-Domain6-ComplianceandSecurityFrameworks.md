# KCSA Exam Study Guide: Domain 6 - Compliance and Security Frameworks

## Domain Weight: 10% (~6 questions out of 60)

This domain covers compliance frameworks, threat modeling frameworks, supply chain compliance, and security automation tools.

---

## 6.1 Compliance Frameworks

### Overview of Compliance

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLIANCE LANDSCAPE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REGULATORY                    INDUSTRY STANDARDS               │
│  ├─ GDPR (EU Privacy)         ├─ CIS Benchmarks                 │
│  ├─ HIPAA (Healthcare)        ├─ NIST Frameworks                │
│  ├─ PCI DSS (Payment Cards)   ├─ SOC 2                          │
│  ├─ SOX (Financial)           ├─ ISO 27001                      │
│  └─ FedRAMP (US Gov)          └─ CSA (Cloud Security Alliance)  │
│                                                                 │
│  KUBERNETES-SPECIFIC                                            │
│  ├─ CIS Kubernetes Benchmark                                    │
│  ├─ NSA/CISA Kubernetes Hardening Guide                         │
│  └─ OWASP Kubernetes Top 10                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### CIS Kubernetes Benchmark

**What is it?**
- Industry-standard security configuration guide
- Covers control plane, worker nodes, and policies
- Scored (pass/fail) and non-scored recommendations
- Regular updates with Kubernetes releases

**CIS Benchmark Categories**:
| Section | Focus Area |
|---------|------------|
| 1. Control Plane Components | API server, controller manager, scheduler, etcd |
| 2. etcd | etcd security configurations |
| 3. Control Plane Configuration | Auth, logging, admission |
| 4. Worker Nodes | kubelet, kube-proxy configuration |
| 5. Policies | Pod security, network policies, secrets, RBAC |

**Example CIS Controls**:
```
1.1.1  - Ensure API server --anonymous-auth is set to false (Scored)
1.2.1  - Ensure --audit-log-path is set (Scored)
1.2.6  - Ensure --kubelet-certificate-authority is set (Scored)
4.2.1  - Ensure --anonymous-auth is set to false on kubelet (Scored)
5.1.1  - Ensure RBAC is enabled (Scored)
5.2.2  - Minimize admission of privileged containers (Scored)
5.2.3  - Minimize admission of containers with allowPrivilegeEscalation (Scored)
5.3.2  - Ensure that all Namespaces have Network Policies defined (Scored)
```

### NIST Frameworks

**NIST Cybersecurity Framework (CSF)**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NIST CSF FUNCTIONS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  IDENTIFY ──► PROTECT ──► DETECT ──► RESPOND ──► RECOVER                │
│                                                                         │
│  • Asset        • Access       • Anomaly    • Response   • Recovery.    │
│    Management     Control        Detection    Planning     Planning.    │
│  • Risk         • Data         • Continuous • Comms      • Improvements │
│    Assessment    Security       Monitoring   • Mitigation               │
│  • Governance   • Training     • Detection  • Analysis                  │
│                                  Processes                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**NIST SP 800-190: Container Security Guide**:
- Image vulnerabilities and countermeasures
- Registry security
- Orchestrator security
- Container runtime security
- Host OS security

**NIST SP 800-204: Microservices Security**:
- Service mesh security
- API security
- DevSecOps practices

### PCI DSS (Payment Card Industry Data Security Standard)

**Relevant Requirements for Kubernetes**:
| Requirement | Description | K8s Implementation |
|-------------|-------------|-------------------|
| 1 | Network segmentation | Network Policies |
| 2 | Secure configurations | CIS Benchmarks, hardening |
| 3 | Protect stored data | Encryption at rest |
| 4 | Encrypt transmission | TLS, mTLS (service mesh) |
| 6 | Secure development | DevSecOps, scanning |
| 7 | Access control | RBAC |
| 8 | Authentication | Strong authentication |
| 10 | Logging and monitoring | Audit logs, Falco |
| 11 | Regular testing | Vulnerability scanning |

### HIPAA (Healthcare)

**Key Controls for Kubernetes**:
- Encryption of PHI (Protected Health Information)
- Access controls and audit logging
- Integrity controls
- Transmission security
- Network segmentation

### GDPR (General Data Protection Regulation)

**Kubernetes Considerations**:
- Data encryption (at rest and in transit)
- Access controls and audit trails
- Data retention and deletion policies
- Cross-border data transfer restrictions
- Right to be forgotten implementation

---

## 6.2 Threat Modeling Frameworks

### STRIDE Framework

**Mnemonic**: STRIDE represents six threat categories

| Letter | Threat | Description | Mitigation |
|--------|--------|-------------|------------|
| **S** | Spoofing | Impersonating another user/system | Authentication |
| **T** | Tampering | Modifying data or code | Integrity checks, signing |
| **R** | Repudiation | Denying actions were taken | Audit logging |
| **I** | Information Disclosure | Exposing sensitive data | Encryption, access control |
| **D** | Denial of Service | Making system unavailable | Rate limiting, quotas |
| **E** | Elevation of Privilege | Gaining unauthorized access | Least privilege, RBAC |

**STRIDE in Kubernetes Context**:
```
┌──────────────────────────────────────────────────────────────────┐
│                 STRIDE for Kubernetes                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SPOOFING                                                        │
│  • Stolen service account tokens                                 │
│  • Fake identity in requests                                     │
│  Mitigation: Strong authentication, short-lived tokens           │
│                                                                  │
│  TAMPERING                                                       │
│  • Modified container images                                     │
│  • Changed ConfigMaps/Secrets                                    │
│  Mitigation: Image signing, RBAC, admission control              │
│                                                                  │
│  REPUDIATION                                                     │
│  • Denying malicious actions                                     │
│  Mitigation: Comprehensive audit logging                         │
│                                                                  │
│  INFORMATION DISCLOSURE                                          │
│  • Exposed secrets, etcd data                                    │
│  • Network sniffing                                              │
│  Mitigation: Encryption, mTLS, Network Policies                  │
│                                                                  │
│  DENIAL OF SERVICE                                               │
│  • Resource exhaustion                                           │
│  • API server flooding                                           │
│  Mitigation: ResourceQuotas, LimitRanges, rate limiting          │
│                                                                  │
│  ELEVATION OF PRIVILEGE                                          │
│  • Container escape                                              │
│  • Over-permissive RBAC                                          │
│  Mitigation: Pod Security Standards, least privilege             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### DREAD Framework

**Risk Rating**: Each threat is scored 1-10 on these factors

| Factor | Question | Example (High Score) |
|--------|----------|----------------------|
| **D**amage | How much damage can occur? | Data breach, system down |
| **R**eproducibility | How easy to reproduce? | Always works |
| **E**xploitability | How easy to exploit? | Script kiddie can do it |
| **A**ffected Users | How many affected? | All users |
| **D**iscoverability | How easy to discover? | Publicly known |

**Risk Score** = (D + R + E + A + D) / 5

| Score Range | Risk Level |
|-------------|------------|
| 1-4 | Low |
| 5-7 | Medium |
| 8-10 | High |

### PASTA (Process for Attack Simulation and Threat Analysis)

**Seven Stages**:
```
┌──────────────────────────────────────────────────────────────────┐
│                    PASTA Framework                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Stage 1: Define Objectives                                      │
│  └─► Business objectives, security requirements                  │
│                                                                  │
│  Stage 2: Define Technical Scope                                 │
│  └─► Architecture, technologies, data flows                      │
│                                                                  │
│  Stage 3: Application Decomposition                              │
│  └─► Components, trust boundaries, entry points                  │
│                                                                  │
│  Stage 4: Threat Analysis                                        │
│  └─► Identify threats using threat intelligence                  │
│                                                                  │
│  Stage 5: Vulnerability Analysis                                 │
│  └─► Identify weaknesses and vulnerabilities                     │
│                                                                  │
│  Stage 6: Attack Modeling                                        │
│  └─► Build attack trees, simulate attacks                        │
│                                                                  │
│  Stage 7: Risk and Impact Analysis                               │
│  └─► Calculate risk, prioritize remediation                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### OCTAVE (Operationally Critical Threat, Asset, and Vulnerability Evaluation)

**Three Phases**:
1. **Organizational View**: Identify assets, threats, vulnerabilities
2. **Technological View**: Identify infrastructure vulnerabilities
3. **Strategy Development**: Develop protection strategy and mitigation plans

### MITRE ATT&CK for Containers

**Key Tactics**:
| Tactic | Description | K8s Examples |
|--------|-------------|--------------|
| Initial Access | Gaining entry | Exposed API, compromised image |
| Execution | Running code | Exec into container, malicious workload |
| Persistence | Maintaining access | CronJob, DaemonSet, webhook |
| Privilege Escalation | Gaining privileges | Privileged pod, hostPath |
| Defense Evasion | Avoiding detection | Clear logs, masquerade names |
| Credential Access | Stealing credentials | SA tokens, secrets |
| Discovery | Reconnaissance | API enumeration, network scan |
| Lateral Movement | Moving in cluster | Service account abuse |
| Collection | Gathering data | Volume access |
| Impact | Causing damage | Resource hijacking, data destruction |

---

## 6.3 Supply Chain Compliance

### Software Supply Chain Security

```
┌─────────────────────────────────────────────────────────────────┐
│               SUPPLY CHAIN COMPLIANCE CONTROLS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CODE                                                           │
│  ✓ Signed commits (GPG)                                         │
│  ✓ Code review required                                         │
│  ✓ Branch protection                                            │
│  ✓ SAST scanning                                                │
│                                                                 │
│  DEPENDENCIES                                                   │
│  ✓ Dependency scanning (SCA)                                    │
│  ✓ Lock files (package-lock.json, go.sum)                       │
│  ✓ Private package mirrors                                      │
│  ✓ Vulnerability tracking                                       │
│                                                                 │
│  BUILD                                                          │
│  ✓ Isolated build environment                                   │
│  ✓ Reproducible builds                                          │
│  ✓ SBOM generation                                              │
│  ✓ Build provenance                                             │
│                                                                 │
│  ARTIFACTS                                                      │
│  ✓ Image signing (Cosign)                                       │
│  ✓ Vulnerability scanning                                       │
│  ✓ Private registries                                           │
│  ✓ Image pinning (digests)                                      │
│                                                                 │
│  DEPLOYMENT                                                     │
│  ✓ Signature verification                                       │
│  ✓ Admission control policies                                   │
│  ✓ GitOps practices                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### SLSA Framework (Supply-chain Levels for Software Artifacts)

| Level | Requirements | Trust |
|-------|--------------|-------|
| **SLSA 1** | Documentation of build process | Low |
| **SLSA 2** | Tamper-resistant build, hosted build service | Medium |
| **SLSA 3** | Hardened builds, auditable provenance | High |
| **SLSA 4** | Two-party review, hermetic builds | Very High |

### Image Signing for Compliance

**Enforce Signed Images with Kyverno**:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "myregistry.io/*"
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
```

### Software Bill of Materials (SBOM)

**Purpose**:
- Complete inventory of software components
- Enables vulnerability tracking
- Supports compliance requirements
- Required by Executive Order 14028 (US Gov)

**Formats**:
- **SPDX**: Linux Foundation standard
- **CycloneDX**: OWASP standard
- **SWID**: ISO standard

---

## 6.4 Automation and Tooling

### Compliance Scanning Tools

| Tool | Purpose | Framework Support |
|------|---------|-------------------|
| **kube-bench** | CIS Benchmark scanning | CIS |
| **Kubescape** | Multi-framework scanning | NSA, CIS, MITRE |
| **Trivy** | Vulnerabilities + misconfig | CIS, custom |
| **Polaris** | Best practices validation | Custom checks |
| **Checkov** | IaC security scanning | CIS, custom |
| **Terrascan** | IaC compliance | Multiple |
| **Falco** | Runtime threat detection | Custom rules |

### kube-bench

**Purpose**: Checks cluster against CIS Kubernetes Benchmark

```bash
# Run on master node
kube-bench run --targets=master

# Run on worker node
kube-bench run --targets=node

# Run all checks
kube-bench run

# Output as JSON
kube-bench run --json > results.json

# Run specific checks
kube-bench run --check 1.1.1,1.1.2
```

**Sample Output**:
```
[INFO] 1 Control Plane Components
[PASS] 1.1.1 Ensure API server --anonymous-auth is set to false
[FAIL] 1.1.2 Ensure API server --audit-log-path is set
[WARN] 1.1.3 Ensure API server --audit-log-maxage is set to 30 or more

== Summary total ==
42 checks PASS
3 checks FAIL
5 checks WARN
0 checks INFO
```

### Kubescape

**Purpose**: Multi-framework security scanning

```bash
# Scan against NSA framework
kubescape scan framework nsa

# Scan against CIS benchmark
kubescape scan framework cis-v1.23-t1.0.1

# Scan against MITRE ATT&CK
kubescape scan framework mitre

# Scan all frameworks
kubescape scan framework all

# Scan specific namespace
kubescape scan --include-namespaces production

# Scan YAML files
kubescape scan *.yaml

# Set failure threshold
kubescape scan framework nsa --compliance-threshold 80
```

### Trivy Configuration Scanning

```bash
# Scan Kubernetes manifests
trivy config ./manifests/

# Scan with specific severity
trivy config --severity HIGH,CRITICAL ./manifests/

# Scan running cluster
trivy k8s cluster --report summary

# Scan specific namespace
trivy k8s --namespace production all
```

### Polaris

**Purpose**: Kubernetes best practices validation

```bash
# Dashboard mode
polaris dashboard --port 8080

# Audit mode (CLI)
polaris audit --format yaml

# Audit specific namespace
polaris audit --namespace production

# Webhook mode (admission controller)
polaris webhook
```

**Default Polaris Checks**:
- Container should not run as root
- Memory limits should be set
- CPU limits should be set
- Liveness probe should be configured
- Readiness probe should be configured
- Images should not use latest tag
- Container should not allow privilege escalation

---

## 6.5 Continuous Security Assessment

### Security Assessment Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│             CONTINUOUS SECURITY ASSESSMENT                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐      │
│  │  ASSESS  │──►│ ANALYZE  │──►│ REMEDIATE│──►│  VERIFY  │      │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘      │
│       │              │              │              │            │
│       ▼              ▼              ▼              ▼            │
│  • Scan          • Prioritize   • Fix issues   • Re-scan        │
│    clusters        by risk        • Update       • Validate     │
│  • Audit logs   • Identify        configs       • Document      │
│  • Review         root cause    • Patch         • Report        │
│    configs                        systems                       │
│       │                                              │          │
│       └──────────────────────────────────────────────┘          │
│                         (Continuous Loop)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### CI/CD Security Pipeline

```yaml
# Example: GitHub Actions compliance pipeline
name: Security Compliance

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    # Scan Kubernetes manifests with Trivy
    - name: Trivy Config Scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'config'
        scan-ref: './k8s/'
        exit-code: '1'
        severity: 'HIGH,CRITICAL'
    
    # Scan with Kubescape
    - name: Kubescape Scan
      uses: kubescape/github-action@main
      with:
        framework: nsa
        failedThreshold: 50
    
    # Scan container image
    - name: Trivy Image Scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        exit-code: '1'
        severity: 'HIGH,CRITICAL'
```

### Compliance Reporting

**Key Metrics**:
| Metric | Description |
|--------|-------------|
| Compliance Score | % of passed checks |
| Critical Findings | Number of critical issues |
| Time to Remediate | Average fix time |
| Policy Violations | Number of admission denials |
| Vulnerability Count | CVEs by severity |

---

## Key Exam Tips for This Domain

1. **Know the frameworks**: CIS, NIST, PCI DSS, HIPAA, GDPR
2. **STRIDE memorization**: Spoofing, Tampering, Repudiation, Information Disclosure, DoS, Elevation of Privilege
3. **kube-bench** scans against CIS Benchmark
4. **Kubescape** supports multiple frameworks (NSA, CIS, MITRE)
5. **SLSA levels**: Higher level = more secure supply chain
6. **SBOM** = Software Bill of Materials
7. **Continuous compliance** = automated scanning in CI/CD

---

## Practice Questions

1. What does STRIDE stand for?
   - Answer: Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege

2. Which tool scans Kubernetes clusters against CIS Benchmark?
   - Answer: kube-bench

3. What compliance framework is required for payment card processing?
   - Answer: PCI DSS

4. What does SBOM stand for?
   - Answer: Software Bill of Materials

5. Which NIST publication covers container security?
   - Answer: NIST SP 800-190

6. What are the five functions of the NIST Cybersecurity Framework?
   - Answer: Identify, Protect, Detect, Respond, Recover

7. Which tool can scan against NSA, CIS, and MITRE frameworks?
   - Answer: Kubescape

8. What does SLSA stand for?
   - Answer: Supply-chain Levels for Software Artifacts

---

## Additional Resources

- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [NIST SP 800-190 Container Security](https://csrc.nist.gov/publications/detail/sp/800-190/final)
- [NSA/CISA Kubernetes Hardening Guide](https://www.cisa.gov/news-events/alerts/2022/03/15/updated-kubernetes-hardening-guide)
- [OWASP Kubernetes Top 10](https://owasp.org/www-project-kubernetes-top-ten/)
- [SLSA Framework](https://slsa.dev/)
- [kube-bench](https://github.com/aquasecurity/kube-bench)
- [Kubescape](https://github.com/kubescape/kubescape)