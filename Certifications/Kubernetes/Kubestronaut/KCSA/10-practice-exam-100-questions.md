# KCSA Practice Exam - 100+ Questions with Answers

## Instructions
- Real exam: 60 questions in 90 minutes
- Passing score: 75% (45 correct)
- For 100%: Master ALL concepts
- Practice until you score 95%+ consistently

---

# DOMAIN 1: OVERVIEW OF CLOUD NATIVE SECURITY (14%)

### Q1. What are the 4Cs of Cloud Native Security (from outer to inner)?
- A) Code, Container, Cluster, Cloud
- B) Cloud, Cluster, Container, Code ✓
- C) Cloud, Container, Cluster, Code
- D) Code, Cluster, Container, Cloud

**Explanation**: The 4Cs from outermost to innermost are Cloud, Cluster, Container, Code.

---

### Q2. Which layer of the 4Cs is the responsibility of the cloud provider in a managed Kubernetes service?
- A) Code
- B) Container
- C) Cluster
- D) Cloud ✓

**Explanation**: The Cloud layer (infrastructure) is primarily the cloud provider's responsibility in a shared responsibility model.

---

### Q3. What is the OWASP Kubernetes Top 10?
- A) A list of top 10 Kubernetes features
- B) A prioritized list of security risks for Kubernetes ✓
- C) A list of top 10 Kubernetes tools
- D) A benchmark for Kubernetes performance

**Explanation**: OWASP Kubernetes Top 10 identifies and prioritizes the most critical security risks in Kubernetes environments.

---

### Q4. Which type of security testing analyzes source code without executing it?
- A) DAST
- B) SAST ✓
- C) IAST
- D) Penetration Testing

**Explanation**: SAST (Static Application Security Testing) analyzes code without execution.

---

### Q5. What is a distroless image?
- A) An image without any operating system
- B) A minimal image without package managers or shells ✓
- C) An image without any applications
- D) An encrypted container image

**Explanation**: Distroless images contain only the application and its runtime dependencies, no shell or package manager.

---

### Q6. Which CNCF project is used for runtime threat detection?
- A) Prometheus
- B) Falco ✓
- C) Trivy
- D) OPA

**Explanation**: Falco is the CNCF graduated project for runtime security and threat detection.

---

### Q7. What does SCA stand for in application security?
- A) Static Code Analysis
- B) Software Composition Analysis ✓
- C) Security Compliance Audit
- D) Source Code Authentication

**Explanation**: SCA (Software Composition Analysis) scans dependencies for known vulnerabilities.

---

### Q8. Which is NOT a valid isolation technique in Kubernetes?
- A) Namespaces
- B) Network Policies
- C) Pod Security Standards
- D) ConfigMaps ✓

**Explanation**: ConfigMaps are for configuration data, not isolation. The others are valid isolation techniques.

---

# DOMAIN 2: KUBERNETES CLUSTER COMPONENT SECURITY (22%)

### Q9. Which component is the "front door" to the Kubernetes cluster?
- A) kubelet
- B) kube-apiserver ✓
- C) etcd
- D) kube-scheduler

**Explanation**: The kube-apiserver is the front-end to the control plane and handles all API requests.

---

### Q10. Where is all Kubernetes cluster state stored?
- A) kube-apiserver
- B) etcd ✓
- C) kubelet
- D) ConfigMaps

**Explanation**: etcd is the distributed key-value store that contains all cluster data.

---

### Q11. What port should be disabled on the kubelet for security?
- A) 10250
- B) 10255 ✓
- C) 6443
- D) 2379

**Explanation**: Port 10255 is the read-only kubelet port and should be disabled (`--read-only-port=0`).

---

### Q12. Which flag enables RBAC authorization on the API server?
- A) --enable-rbac=true
- B) --authorization-mode=RBAC ✓
- C) --rbac-enabled
- D) --auth-mode=RBAC

**Explanation**: `--authorization-mode=RBAC` enables Role-Based Access Control.

---

### Q13. What encryption should be used for etcd peer communication?
- A) TLS
- B) mTLS ✓
- C) SSH
- D) IPSec

**Explanation**: mTLS (Mutual TLS) should be used for etcd peer-to-peer communication.

---

### Q14. Which kubelet flag protects kernel defaults?
- A) --secure-kernel=true
- B) --protect-kernel-defaults=true ✓
- C) --kernel-protection=enabled
- D) --secure-defaults=true

**Explanation**: `--protect-kernel-defaults=true` ensures kubelet doesn't modify kernel parameters.

---

### Q15. Which container runtime replaced Docker as the default in Kubernetes 1.24+?
- A) CRI-O
- B) containerd ✓
- C) runc
- D) podman

**Explanation**: containerd became the default runtime after dockershim was removed in Kubernetes 1.24.

---

### Q16. What is the default network policy behavior in Kubernetes?
- A) Deny all traffic
- B) Allow all traffic ✓
- C) Allow only DNS traffic
- D) Allow only same-namespace traffic

**Explanation**: By default, all pod-to-pod traffic is allowed. Network Policies must explicitly restrict traffic.

---

### Q17. Which volume type should be avoided in production due to security risks?
- A) PersistentVolumeClaim
- B) emptyDir
- C) hostPath ✓
- D) ConfigMap

**Explanation**: hostPath mounts host filesystem into pods, creating significant security risks.

---

### Q18. What permissions should be set on kubeconfig files?
- A) 644
- B) 755
- C) 600 ✓
- D) 777

**Explanation**: kubeconfig should be readable only by the owner (600 or rw-------).

---

# DOMAIN 3: KUBERNETES SECURITY FUNDAMENTALS (22%)

### Q19. What are the three Pod Security Standards levels?
- A) Low, Medium, High
- B) Privileged, Baseline, Restricted ✓
- C) Permissive, Default, Strict
- D) Open, Limited, Locked

**Explanation**: Pod Security Standards define Privileged (unrestricted), Baseline (minimal), and Restricted (hardened).

---

### Q20. What replaced PodSecurityPolicy in Kubernetes 1.25?
- A) SecurityContext
- B) Pod Security Admission ✓
- C) NetworkPolicy
- D) RBAC

**Explanation**: Pod Security Admission (PSA) replaced the deprecated PodSecurityPolicy in Kubernetes 1.25.

---

### Q21. Which PSA mode rejects non-compliant pods?
- A) audit
- B) warn
- C) enforce ✓
- D) deny

**Explanation**: The `enforce` mode rejects pods that violate the security standard.

---

### Q22. Are Kubernetes Secrets encrypted by default?
- A) Yes, with AES-256
- B) Yes, with RSA
- C) No, they are base64 encoded ✓
- D) No, they are stored in plaintext

**Explanation**: Secrets are base64 ENCODED, not encrypted. Encryption at rest must be configured separately.

---

### Q23. What is the difference between Role and ClusterRole?
- A) Role is for users, ClusterRole is for services
- B) Role is namespace-scoped, ClusterRole is cluster-wide ✓
- C) Role is read-only, ClusterRole has write access
- D) There is no difference

**Explanation**: Role applies within a namespace; ClusterRole applies cluster-wide or across namespaces.

---

### Q24. Which RBAC verbs are typically used for read-only access?
- A) create, update, delete
- B) get, list, watch ✓
- C) read, view, access
- D) fetch, retrieve, observe

**Explanation**: The verbs `get`, `list`, and `watch` provide read-only access to resources.

---

### Q25. What securityContext option prevents privilege escalation?
- A) runAsNonRoot: true
- B) readOnlyRootFilesystem: true
- C) allowPrivilegeEscalation: false ✓
- D) privileged: false

**Explanation**: `allowPrivilegeEscalation: false` prevents processes from gaining more privileges than their parent.

---

### Q26. What should automountServiceAccountToken be set to when the token is not needed?
- A) true
- B) false ✓
- C) disabled
- D) null

**Explanation**: Set `automountServiceAccountToken: false` when the service account token is not needed.

---

### Q27. Which Kubernetes audit log level records the request body?
- A) None
- B) Metadata
- C) Request ✓
- D) RequestResponse

**Explanation**: `Request` level logs metadata plus the request body. `RequestResponse` also includes the response body.

---

### Q28. What does a NetworkPolicy with empty podSelector ({}) select?
- A) No pods
- B) All pods in the namespace ✓
- C) Only pods without labels
- D) Only system pods

**Explanation**: An empty podSelector `{}` selects all pods in the namespace.

---

### Q29. Which authentication method is recommended for human users in enterprise environments?
- A) X.509 certificates
- B) Static tokens
- C) OIDC (OpenID Connect) ✓
- D) Bootstrap tokens

**Explanation**: OIDC integrates with enterprise identity providers for human user authentication.

---

### Q30. What happens when a user bypasses RBAC as a member of system:masters group?
- A) They get limited admin access
- B) They get full unrestricted access ✓
- C) They are denied access
- D) They get read-only access

**Explanation**: system:masters group bypasses ALL RBAC checks and provides unrestricted superuser access.

---

# DOMAIN 4: KUBERNETES THREAT MODEL (16%)

### Q31. What IP address is the cloud metadata API that should be blocked?
- A) 127.0.0.1
- B) 10.0.0.1
- C) 169.254.169.254 ✓
- D) 192.168.1.1

**Explanation**: 169.254.169.254 is the cloud metadata service that can leak sensitive information.

---

### Q32. Which pod configuration gives full access to the host system?
- A) hostNetwork: true
- B) hostPID: true
- C) privileged: true ✓
- D) hostIPC: true

**Explanation**: `privileged: true` gives the container almost all host system capabilities.

---

### Q33. What type of attack maintains access after initial compromise?
- A) Denial of Service
- B) Persistence ✓
- C) Initial Access
- D) Discovery

**Explanation**: Persistence attacks maintain access long after the initial compromise.

---

### Q34. Which Kubernetes resource can be used for persistence via scheduled tasks?
- A) DaemonSet
- B) Deployment
- C) CronJob ✓
- D) StatefulSet

**Explanation**: CronJobs can be used by attackers to schedule persistent malicious tasks.

---

### Q35. What framework categorizes adversary tactics and techniques?
- A) CIS Benchmark
- B) NIST CSF
- C) MITRE ATT&CK ✓
- D) OWASP Top 10

**Explanation**: MITRE ATT&CK provides a comprehensive matrix of adversary tactics and techniques.

---

### Q36. What type of attack involves similar package names to trick developers?
- A) SQL Injection
- B) Typosquatting ✓
- C) Cross-site scripting
- D) Buffer overflow

**Explanation**: Typosquatting uses similar-looking package names (e.g., `lodash` vs `1odash`).

---

### Q37. Which attack allows an attacker to escape from a container to the host?
- A) SQL Injection
- B) Container Escape ✓
- C) DNS Spoofing
- D) Session Hijacking

**Explanation**: Container escape exploits allow breaking out of container isolation to access the host.

---

### Q38. What prevents resource exhaustion DoS attacks?
- A) NetworkPolicies
- B) ResourceQuotas and LimitRanges ✓
- C) RBAC
- D) Secrets

**Explanation**: ResourceQuotas and LimitRanges limit resource consumption per namespace/pod.

---

### Q39. What is the primary target for credential theft in Kubernetes?
- A) ConfigMaps
- B) etcd ✓
- C) Deployments
- D) Services

**Explanation**: etcd contains all cluster data including secrets in accessible form.

---

### Q40. Which capability should be dropped from containers for security?
- A) NET_BIND_SERVICE
- B) ALL ✓
- C) CHOWN
- D) SETUID

**Explanation**: Best practice is to drop ALL capabilities and add only what's needed.

---

# DOMAIN 5: PLATFORM SECURITY (16%)

### Q41. What does SBOM stand for?
- A) Security Bill of Materials
- B) Software Bill of Materials ✓
- C) System Build Object Model
- D) Secure Binary Object Manifest

**Explanation**: SBOM (Software Bill of Materials) is a complete list of software components.

---

### Q42. Which tool is used for signing container images?
- A) Trivy
- B) Falco
- C) Cosign ✓
- D) kube-bench

**Explanation**: Cosign (from Sigstore) is used for signing and verifying container images.

---

### Q43. What does mTLS provide in a service mesh?
- A) Only encryption
- B) Only authentication
- C) Mutual authentication and encryption ✓
- D) Only authorization

**Explanation**: mTLS provides both mutual (client and server) authentication AND encryption.

---

### Q44. Which proxy does Istio use in its data plane?
- A) NGINX
- B) HAProxy
- C) Envoy ✓
- D) Traefik

**Explanation**: Istio uses Envoy proxy as its sidecar in the data plane.

---

### Q45. Which CNCF project automates certificate management?
- A) Falco
- B) cert-manager ✓
- C) Prometheus
- D) Jaeger

**Explanation**: cert-manager automates the management and issuance of TLS certificates.

---

### Q46. What is the order of admission controller processing?
- A) Validating then Mutating
- B) Mutating then Validating ✓
- C) Simultaneous
- D) Random

**Explanation**: Mutating admission webhooks run first, then Validating webhooks.

---

### Q47. Which policy engine uses the Rego language?
- A) Kyverno
- B) OPA (Open Policy Agent) ✓
- C) Polaris
- D) Falco

**Explanation**: OPA uses Rego, a purpose-built policy language.

---

### Q48. What type of base image has the smallest attack surface?
- A) Ubuntu
- B) Alpine
- C) Distroless ✓
- D) CentOS

**Explanation**: Distroless images contain only the application and runtime, no shell or package manager.

---

### Q49. Which tool is used for runtime syscall monitoring?
- A) Trivy
- B) kube-bench
- C) Falco ✓
- D) Kubescape

**Explanation**: Falco monitors syscalls and kernel events for security threats.

---

### Q50. What does Harbor provide that Docker Hub doesn't by default?
- A) Image storage
- B) Built-in vulnerability scanning and RBAC ✓
- C) Container runtime
- D) Kubernetes deployment

**Explanation**: Harbor is a CNCF project providing vulnerability scanning, signing, and RBAC for registries.

---

# DOMAIN 6: COMPLIANCE AND SECURITY FRAMEWORKS (10%)

### Q51. What does STRIDE stand for?
- A) Security, Trust, Risk, Identity, Defense, Encryption
- B) Spoofing, Tampering, Repudiation, Information Disclosure, DoS, Elevation of Privilege ✓
- C) System, Threat, Response, Investigation, Detection, Evaluation
- D) Secure, Trusted, Reliable, Isolated, Defended, Encrypted

**Explanation**: STRIDE is a threat modeling framework with six threat categories.

---

### Q52. Which tool scans Kubernetes clusters against CIS Benchmark?
- A) Trivy
- B) kube-bench ✓
- C) Falco
- D) Polaris

**Explanation**: kube-bench specifically checks CIS Kubernetes Benchmark compliance.

---

### Q53. What compliance framework is required for payment card processing?
- A) HIPAA
- B) GDPR
- C) PCI DSS ✓
- D) SOX

**Explanation**: PCI DSS (Payment Card Industry Data Security Standard) is required for payment card data.

---

### Q54. Which NIST publication covers container security?
- A) NIST SP 800-53
- B) NIST SP 800-190 ✓
- C) NIST SP 800-171
- D) NIST SP 800-63

**Explanation**: NIST SP 800-190 is the Application Container Security Guide.

---

### Q55. What are the five functions of the NIST Cybersecurity Framework?
- A) Plan, Do, Check, Act, Review
- B) Identify, Protect, Detect, Respond, Recover ✓
- C) Assess, Implement, Monitor, Report, Improve
- D) Design, Build, Test, Deploy, Operate

**Explanation**: NIST CSF functions are Identify, Protect, Detect, Respond, and Recover.

---

### Q56. Which tool can scan against NSA, CIS, and MITRE frameworks?
- A) kube-bench
- B) Kubescape ✓
- C) Trivy
- D) Polaris

**Explanation**: Kubescape supports multiple frameworks including NSA, CIS, and MITRE ATT&CK.

---

### Q57. What does SLSA stand for?
- A) Security Levels for Software Applications
- B) Supply-chain Levels for Software Artifacts ✓
- C) Standard Levels for Secure Architecture
- D) System Levels for Security Assessment

**Explanation**: SLSA defines supply chain security levels for software artifacts.

---

### Q58. What is DREAD used for?
- A) Threat categorization
- B) Risk rating ✓
- C) Compliance auditing
- D) Vulnerability scanning

**Explanation**: DREAD is a risk rating methodology (Damage, Reproducibility, Exploitability, Affected users, Discoverability).

---

### Q59. Which compliance framework applies to healthcare data in the US?
- A) PCI DSS
- B) GDPR
- C) HIPAA ✓
- D) SOC 2

**Explanation**: HIPAA (Health Insurance Portability and Accountability Act) protects healthcare data.

---

### Q60. What does PASTA stand for in threat modeling?
- A) Process for Attack Simulation and Threat Analysis ✓
- B) Protocol for Application Security Testing and Assessment
- C) Procedure for Analyzing System Threat Architecture
- D) Plan for Application Security Threat Assessment

**Explanation**: PASTA is a seven-stage threat modeling methodology.

---

# BONUS QUESTIONS (Mixed Domains)

### Q61. Which securityContext setting runs a container as non-root?
- A) privileged: false
- B) runAsNonRoot: true ✓
- C) allowPrivilegeEscalation: false
- D) readOnlyRootFilesystem: true

**Explanation**: `runAsNonRoot: true` ensures the container doesn't run as UID 0.

---

### Q62. What does a default deny NetworkPolicy look like?
- A) podSelector with specific labels
- B) Empty podSelector {} with no ingress/egress rules ✓
- C) podSelector: all
- D) deny: true

**Explanation**: Empty podSelector with policyTypes but no rules creates a deny-all policy.

---

### Q63. Which service mesh is known for being lightweight and using a Rust-based proxy?
- A) Istio
- B) Linkerd ✓
- C) Consul Connect
- D) AWS App Mesh

**Explanation**: Linkerd uses a lightweight Rust-based proxy (linkerd-proxy).

---

### Q64. What is the purpose of image pinning with digests?
- A) Faster image pulls
- B) Ensure immutable, verified images ✓
- C) Smaller image size
- D) Better compression

**Explanation**: Image digests ensure you get exactly the same image every time (immutable).

---

### Q65. Which Kubernetes component should have a separate CA from the main cluster?
- A) API Server
- B) kubelet
- C) etcd ✓
- D) kube-proxy

**Explanation**: etcd should have its own CA for additional isolation.

---

### Q66. What is the main difference between Kyverno and OPA?
- A) Kyverno uses YAML, OPA uses Rego ✓
- B) OPA uses YAML, Kyverno uses Rego
- C) They are identical
- D) Kyverno is for storage, OPA is for networking

**Explanation**: Kyverno uses Kubernetes-native YAML; OPA uses the Rego policy language.

---

### Q67. What does the 'S' in STRIDE represent?
- A) Security
- B) Spoofing ✓
- C) System
- D) Scanning

**Explanation**: S = Spoofing (impersonating another user or system).

---

### Q68. Which admission controller was removed in Kubernetes 1.25?
- A) PodSecurity
- B) PodSecurityPolicy ✓
- C) NodeRestriction
- D) LimitRanger

**Explanation**: PodSecurityPolicy (PSP) was removed in Kubernetes 1.25.

---

### Q69. What is the recommended minimum for ResourceQuota CPU limits?
- A) Based on actual workload needs ✓
- B) Always 1 CPU
- C) No limit needed
- D) 100m for all containers

**Explanation**: Limits should be based on actual workload requirements, not arbitrary values.

---

### Q70. Which tool generates SBOM for container images?
- A) Falco
- B) Syft ✓
- C) kube-bench
- D) Polaris

**Explanation**: Syft generates Software Bill of Materials for container images and filesystems.

---

### Q71. What does the NodeRestriction admission controller do?
- A) Limits pod scheduling
- B) Restricts kubelet's API permissions ✓
- C) Denies all external traffic
- D) Limits resource usage

**Explanation**: NodeRestriction limits kubelet to only modify its own node and pods running on it.

---

### Q72. Which file format is NOT used for SBOM?
- A) SPDX
- B) CycloneDX
- C) YAML ✓
- D) SWID

**Explanation**: SPDX, CycloneDX, and SWID are SBOM formats. YAML is a general format.

---

### Q73. What is the purpose of Seccomp in containers?
- A) Encrypt network traffic
- B) Restrict system calls ✓
- C) Manage secrets
- D) Control CPU usage

**Explanation**: Seccomp (Secure Computing Mode) restricts which syscalls a container can make.

---

### Q74. Which PSS level is most restrictive?
- A) Privileged
- B) Baseline
- C) Restricted ✓
- D) Default

**Explanation**: Restricted is the most secure Pod Security Standard level.

---

### Q75. What does CIS in "CIS Benchmark" stand for?
- A) Container Image Security
- B) Center for Internet Security ✓
- C) Cloud Infrastructure Security
- D) Certified Information Security

**Explanation**: CIS = Center for Internet Security, a non-profit organization.

---

### Q76. Which capability is required to bind to ports below 1024?
- A) CAP_SYS_ADMIN
- B) CAP_NET_BIND_SERVICE ✓
- C) CAP_NET_RAW
- D) CAP_SETUID

**Explanation**: CAP_NET_BIND_SERVICE allows binding to privileged ports (< 1024).

---

### Q77. What is the purpose of AppArmor in Kubernetes?
- A) Application performance monitoring
- B) Mandatory access control for containers ✓
- C) Application deployment
- D) Network monitoring

**Explanation**: AppArmor provides mandatory access control (MAC) for Linux applications.

---

### Q78. Which command checks if a user can perform an action in Kubernetes?
- A) kubectl auth check
- B) kubectl auth can-i ✓
- C) kubectl permissions
- D) kubectl access-check

**Explanation**: `kubectl auth can-i` checks authorization for specific actions.

---

### Q79. What is a sidecar container in a service mesh?
- A) A backup container
- B) A proxy container alongside the application ✓
- C) A logging container
- D) A monitoring container

**Explanation**: Sidecar proxies handle traffic for the application container in service meshes.

---

### Q80. Which encryption provider is recommended for secrets at rest?
- A) identity
- B) aescbc or aesgcm ✓
- C) secretbox
- D) base64

**Explanation**: aescbc or aesgcm provide AES encryption for secrets at rest.

---

### Q81. What does the 'E' in DREAD represent?
- A) Encryption
- B) Exploitability ✓
- C) Evaluation
- D) Execution

**Explanation**: E = Exploitability (how easy is the vulnerability to exploit).

---

### Q82. Which tool is used for Kubernetes policy-as-code with YAML?
- A) OPA
- B) Kyverno ✓
- C) Falco
- D) Trivy

**Explanation**: Kyverno uses native Kubernetes YAML for policies.

---

### Q83. What is the purpose of ValidatingWebhookConfiguration?
- A) Modify incoming requests
- B) Validate and accept/reject requests ✓
- C) Authenticate users
- D) Authorize actions

**Explanation**: Validating webhooks validate requests without modifying them.

---

### Q84. Which component manages certificate signing requests in Kubernetes?
- A) kube-apiserver
- B) kube-controller-manager ✓
- C) kubelet
- D) kube-scheduler

**Explanation**: The certificate controller in kube-controller-manager handles CSRs.

---

### Q85. What is the default seccompProfile type recommended for containers?
- A) Unconfined
- B) RuntimeDefault ✓
- C) Localhost
- D) None

**Explanation**: RuntimeDefault uses the container runtime's default seccomp profile.

---

### Q86. Which Falco output channel sends alerts to a message queue?
- A) stdout
- B) file
- C) grpc ✓
- D) syslog

**Explanation**: gRPC output can be used to send alerts to external systems.

---

### Q87. What does the AlwaysPullImages admission controller do?
- A) Caches all images locally
- B) Forces image pull for every pod creation ✓
- C) Deletes unused images
- D) Compresses images

**Explanation**: AlwaysPullImages ensures fresh images and validates pull credentials.

---

### Q88. Which compliance framework addresses data privacy in the EU?
- A) PCI DSS
- B) HIPAA
- C) GDPR ✓
- D) SOX

**Explanation**: GDPR (General Data Protection Regulation) is the EU data privacy law.

---

### Q89. What is a mutating webhook used for?
- A) Rejecting invalid requests
- B) Modifying requests before persistence ✓
- C) Logging requests
- D) Authenticating users

**Explanation**: Mutating webhooks can modify (mutate) requests before validation.

---

### Q90. Which CNCF project provides workload identity?
- A) Falco
- B) SPIFFE/SPIRE ✓
- C) Prometheus
- D) Jaeger

**Explanation**: SPIFFE/SPIRE provides secure workload identity in cloud-native environments.

---

### Q91. What is the purpose of LimitRange?
- A) Limit network bandwidth
- B) Set default and limits for resources in namespace ✓
- C) Limit number of pods
- D) Limit storage usage

**Explanation**: LimitRange sets default requests/limits for containers in a namespace.

---

### Q92. Which attack vector uses malicious dependencies with similar names?
- A) Dependency Confusion ✓
- B) SQL Injection
- C) XSS
- D) CSRF

**Explanation**: Dependency confusion exploits private vs public package naming.

---

### Q93. What does 'R' in STRIDE represent?
- A) Risk
- B) Repudiation ✓
- C) Response
- D) Recovery

**Explanation**: R = Repudiation (denying that an action was taken).

---

### Q94. Which Kubernetes resource defines storage encryption classes?
- A) PersistentVolume
- B) StorageClass ✓
- C) Secret
- D) ConfigMap

**Explanation**: StorageClass can specify encryption parameters for dynamic provisioning.

---

### Q95. What is the purpose of in-toto?
- A) Container runtime
- B) Supply chain integrity framework ✓
- C) Service mesh
- D) Logging tool

**Explanation**: in-toto is a framework for securing software supply chain integrity.

---

### Q96. Which PSA mode logs violations but allows pods?
- A) enforce
- B) audit ✓
- C) warn
- D) deny

**Explanation**: `audit` mode logs violations to the audit log but allows the pod.

---

### Q97. What is the recommended practice for service account tokens?
- A) Use long-lived tokens
- B) Use short-lived, projected tokens ✓
- C) Share tokens between services
- D) Disable token authentication

**Explanation**: Projected service account tokens with short expiry are more secure.

---

### Q98. Which tool scans container images for vulnerabilities?
- A) kube-bench
- B) Trivy ✓
- C) Kubescape
- D) Polaris

**Explanation**: Trivy scans container images, filesystems, and configs for vulnerabilities.

---

### Q99. What does the DenyServiceExternalIPs admission controller mitigate?
- A) DoS attacks
- B) CVE-2020-8554 (MitM attack) ✓
- C) Container escape
- D) Privilege escalation

**Explanation**: DenyServiceExternalIPs mitigates CVE-2020-8554 man-in-the-middle attack.

---

### Q100. Which log level in Kubernetes audit logs everything including response?
- A) None
- B) Metadata
- C) Request
- D) RequestResponse ✓

**Explanation**: RequestResponse logs metadata, request body, and response body.

---

## Scoring Guide

| Score | Readiness |
|-------|-----------|
| 95-100 | Ready for 100%! |
| 85-94 | Ready to pass, review weak areas |
| 75-84 | Borderline, need more study |
| Below 75 | Significant preparation needed |

**Target: Score 95%+ before taking the real KCSA exam.**