# KCSA Exam Study Guide: Domain 4 - Kubernetes Threat Model

## Domain Weight: 16% (~10 questions out of 60)

This domain covers understanding Kubernetes threat landscape, attack vectors, trust boundaries, and common attack techniques.

---

## 4.1 Kubernetes Trust Boundaries

### Trust Boundary Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL ZONE                              │
│                     (Internet, Untrusted Networks)                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      CLUSTER BOUNDARY                       │    │
│  │                                                             │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │                CONTROL PLANE ZONE                   │    │    │
│  │  │   ┌──────────┐  ┌──────────┐  ┌──────────────┐      │    │    │
│  │  │   │API Server│  │   etcd   │  │ Controllers  │      │    │    │
│  │  │   └──────────┘  └──────────┘  └──────────────┘      │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  │                           │                                 │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │                  WORKER NODE ZONE                   │    │    │
│  │  │   ┌──────────┐  ┌──────────┐  ┌──────────────┐      │    │    │
│  │  │   │ kubelet  │  │kube-proxy│  │  Runtime     │      │    │    │
│  │  │   └──────────┘  └──────────┘  └──────────────┘      │    │    │
│  │  │                                                     │    │    │
│  │  │   ┌─────────────────────────────────────────────┐   │    │    │
│  │  │   │              CONTAINER ZONE                 │   │    │    │
│  │  │   │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐         │   │    │    │
│  │  │   │  │ Pod │  │ Pod │  │ Pod │  │ Pod │         │   │    │    │
│  │  │   │  └─────┘  └─────┘  └─────┘  └─────┘         │   │    │    │
│  │  │   └─────────────────────────────────────────────┘   │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Trust Zones

| Zone | Trust Level | Components | Security Focus |
|------|-------------|------------|----------------|
| **External** | Untrusted | Internet, external users | Firewalls, ingress control |
| **Cluster Boundary** | Semi-trusted | API access, node network | Network policies, AuthN |
| **Control Plane** | Highly trusted | API server, etcd, controllers | Strong authentication, encryption |
| **Worker Node** | Trusted | kubelet, runtime | Node hardening, isolation |
| **Container** | Least trusted | Application workloads | Pod security, RBAC |

### Data Flow Security

```
User Request → Load Balancer → Ingress → Service → Pod
                    ↓
              [TLS Termination]
                    ↓
              [Authentication]
                    ↓
              [Authorization]
                    ↓
              [Admission Control]
                    ↓
              [Network Policy]
```

---

## 4.2 Threat Actors and Attack Vectors

### Threat Actor Types

| Actor | Description | Motivation | Capability |
|-------|-------------|------------|------------|
| **External Attacker** | Outside the organization | Financial gain, disruption | Varies |
| **Malicious Insider** | Employee with access | Revenge, financial | High |
| **Compromised User** | Legitimate user, stolen creds | N/A | Depends on role |
| **Compromised Workload** | Exploited container | N/A | Container privileges |
| **Supply Chain Attack** | Malicious dependency | Wide impact | Can be sophisticated |

### Common Attack Vectors

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES ATTACK VECTORS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXTERNAL                                                       │
│  • Exposed API server                                           │
│  • Vulnerable ingress                                           │
│  • Exposed dashboards (K8s dashboard, Grafana)                  │
│  • Exposed etcd                                                 │
│                                                                 │
│  INTERNAL                                                       │
│  • Compromised container → Lateral movement                     │
│  • Stolen credentials/tokens                                    │
│  • Misconfigured RBAC                                           │
│  • Pod-to-pod attacks (no network policy)                       │
│                                                                 │
│  SUPPLY CHAIN                                                   │
│  • Malicious base images                                        │
│  • Compromised dependencies                                     │
│  • Typosquatting attacks                                        │
│  • Compromised CI/CD pipeline                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Common Kubernetes Attacks

### 4.3.1 Privilege Escalation

**Definition**: Gaining higher privileges than originally granted.

**Attack Paths**:
```
┌─────────────────────────────────────────────────────────────────┐
│                   PRIVILEGE ESCALATION PATHS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pod → Node:                                                    │
│  • hostPath volume (mount host filesystem)                      │
│  • hostPID/hostNetwork/hostIPC                                  │
│  • privileged: true                                             │
│  • CAP_SYS_ADMIN capability                                     │
│                                                                 │
│  Pod → Cluster:                                                 │
│  • Service account token theft                                  │
│  • Overly permissive RBAC                                       │
│  • Access to secrets                                            │
│                                                                 │
│  User → Admin:                                                  │
│  • Role escalation via create rolebindings                      │
│  • Impersonation rights                                         │
│  • Controller manager service account                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**"Bad Pods" - Dangerous Pod Configurations**:

| Configuration | Risk Level | Attack |
|---------------|------------|--------|
| `privileged: true` | Critical | Full node access |
| `hostPID: true` | High | See host processes |
| `hostNetwork: true` | High | Access node network |
| `hostPath: /` | Critical | Read/write host filesystem |
| `CAP_SYS_ADMIN` | Critical | Near-root capabilities |
| `automountServiceAccountToken: true` | Medium | Token theft |

### 4.3.2 Persistence Attacks

**Definition**: Maintaining access after initial compromise.

**Persistence Techniques**:

| Technique | Description | Detection |
|-----------|-------------|-----------|
| **Malicious DaemonSet** | Deploy to all nodes | Audit DaemonSet creation |
| **CronJob backdoor** | Scheduled malicious jobs | Review CronJobs |
| **Mutating Webhook** | Inject into all pods | Audit webhook configs |
| **Modified images** | Backdoored container images | Image scanning |
| **Service account abuse** | Create persistent tokens | Token auditing |
| **Static pods** | Direct kubelet manipulation | Node inspection |

### 4.3.3 Denial of Service (DoS)

**Resource Exhaustion Attacks**:

| Target | Attack | Mitigation |
|--------|--------|------------|
| **CPU** | Fork bomb, crypto mining | CPU limits |
| **Memory** | Memory leak, heap spray | Memory limits |
| **Storage** | Fill volumes | Storage quotas |
| **Network** | DDoS, bandwidth exhaustion | Network policies, rate limiting |
| **API Server** | API flooding | Rate limiting, ResourceQuotas |
| **etcd** | Large objects, many writes | Object size limits |

**Protection Measures**:
```yaml
# Resource Quota to prevent DoS
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: default
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```

### 4.3.4 Malicious Code Execution

**Attack Scenarios**:

1. **Container Escape**: Breaking out of container to host
2. **Remote Code Execution (RCE)**: Exploiting application vulnerabilities
3. **Command Injection**: Injecting commands through application input
4. **Cryptomining**: Unauthorized resource usage

**Mitigations**:
- Use minimal base images (distroless)
- Enable Seccomp profiles
- Use read-only root filesystem
- Drop all capabilities
- Run as non-root
- Keep software updated

### 4.3.5 Network Attacks

**Within Cluster**:

| Attack | Description | Mitigation |
|--------|-------------|------------|
| **ARP Spoofing** | Intercept traffic | CNI encryption |
| **DNS Spoofing** | Redirect DNS | Secure CoreDNS |
| **Man-in-the-Middle** | Intercept communication | mTLS (Service Mesh) |
| **Pod-to-Pod sniffing** | Capture traffic | Network policies |
| **Service account theft** | Steal tokens via network | Disable auto-mounting |

**Network Policy Defense**:
```yaml
# Deny all, then allow specific
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 4.3.6 Credential and Data Theft

**Targets**:
- etcd data (all secrets in plaintext)
- Service account tokens
- kubeconfig files
- Cloud provider credentials (metadata API)
- ConfigMaps with sensitive data
- Environment variables

**Cloud Metadata API Attack**:
```bash
# From inside a pod, access cloud metadata
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**Mitigation**: Block metadata API access via Network Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # Block metadata API
```

---

## 4.4 MITRE ATT&CK for Kubernetes

### MITRE ATT&CK Matrix Overview

```
┌────────────────────────────────────────────────────────────────┐
│              MITRE ATT&CK for Kubernetes (Simplified)          │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│Initial Access│ Execution    │ Persistence  │ Privilege         │
│              │              │              │ Escalation        │
├──────────────┼──────────────┼──────────────┼───────────────────┤
│• Exposed API │• Exec into   │• Backdoor    │• Privileged       │
│• Kubeconfig  │  container   │  container   │  container        │
│• Compromised │• New         │• Writable    │• hostPath mount   │
│  image       │  container   │  hostPath    │• Cluster-admin    │
│• App vuln    │• App exploit │• K8s CronJob │  binding          │
└──────────────┴──────────────┴──────────────┴───────────────────┘
┌──────────────┬──────────────┬──────────────┬───────────────────┐
│Defense       │ Credential   │ Discovery    │ Lateral Movement  │
│Evasion       │ Access       │              │                   │
├──────────────┼──────────────┼──────────────┼───────────────────┤
│• Clear logs  │• List K8s    │• Access K8s  │• Access cloud     │
│• Pod/        │  secrets     │  API         │  resources        │
│  container   │• Mount SA    │• Network     │• Container        │
│  name        │  token       │  mapping     │  service account  │
│  similarity  │• App creds   │• Pod listing │• Cluster internal │
│              │  in config   │              │  networking       │
└──────────────┴──────────────┴──────────────┴───────────────────┘
┌──────────────┬──────────────┐
│ Collection   │ Impact       │
├──────────────┼──────────────┤
│• Images from │• Data        │
│  registry    │  destruction │
│• Private     │• Resource    │
│  registry    │  hijacking   │
│              │• DoS         │
└──────────────┴──────────────┘
```

### Key MITRE Techniques for Kubernetes

| ID | Technique | Description |
|----|-----------|-------------|
| T1610 | Deploy Container | Adversary deploys malicious container |
| T1611 | Escape to Host | Container escape to node |
| T1552.007 | Container API | Access Kubernetes API from container |
| T1053.007 | Scheduled Task | CronJob for persistence |
| T1609 | Container Admin | Exec into container |
| T1613 | Container Discovery | Enumerate containers |

---

## 4.5 Supply Chain Attacks

### Supply Chain Attack Surface

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOFTWARE SUPPLY CHAIN                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Code           Build          Distribute       Deploy          │
│  ┌────┐        ┌────┐         ┌─────┐          ┌────┐           │
│  │Dev │───────►│CI/ │────────►│Reg  │─────────►│K8s │           │
│  │IDE │        │CD  │         │istry│          │    │           │
│  └────┘        └────┘         └─────┘          └────┘           │
│    │             │              │               │               │
│    ▼             ▼              ▼               ▼               │
│  • Malicious  • Compromised  • Image         • Compromised      │
│    code         build env      poisoning       admission        │
│  • Typosquat  • Dependency   • Registry      • Malicious        │
│    packages     confusion      compromise      operator         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Supply Chain Attack Types

| Attack | Description | Example |
|--------|-------------|---------|
| **Typosquatting** | Similar package names | `lodash` vs `1odash` |
| **Dependency Confusion** | Private vs public packages | Internal package names |
| **Compromised Dependencies** | Malware in dependencies | event-stream incident |
| **Malicious Base Images** | Backdoored images | Fake nginx image |
| **Registry Compromise** | Attack on registry | Docker Hub incidents |
| **CI/CD Pipeline Attack** | Compromise build system | SolarWinds attack |

### Supply Chain Security Controls

| Control | Description |
|---------|-------------|
| **Image Signing** | Cosign, Notary for image verification |
| **SBOM** | Software Bill of Materials |
| **Vulnerability Scanning** | Trivy, Grype, Snyk |
| **Admission Control** | Only allow signed/scanned images |
| **Private Registry** | Control image sources |
| **Dependency Scanning** | npm audit, pip-audit |

---

## 4.6 Threat Detection and Monitoring

### Detection Strategies

| Layer | What to Monitor | Tools |
|-------|-----------------|-------|
| **API Server** | Audit logs, unusual API calls | Falco, audit logging |
| **Network** | Unusual traffic patterns | Cilium, Calico |
| **Runtime** | Syscalls, process execution | Falco, Sysdig |
| **Container** | File changes, new processes | Falco |
| **Host** | Node-level anomalies | OSSEC, auditd |

### Falco Rules Examples

```yaml
# Detect shell in container
- rule: Terminal shell in container
  desc: Detect shell started in container
  condition: >
    spawned_process and container and
    shell_procs and proc.tty != 0
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name shell=%proc.name)
  priority: WARNING

# Detect sensitive file access
- rule: Read sensitive file
  desc: Detect reading sensitive files
  condition: >
    open_read and sensitive_files and container
  output: >
    Sensitive file opened for reading
    (file=%fd.name container=%container.name)
  priority: WARNING
```

---

## Key Exam Tips for This Domain

1. **Know trust boundaries**: External > Cluster > Control Plane > Node > Container
2. **Understand "Bad Pods"**: privileged, hostPath, hostPID, hostNetwork
3. **MITRE ATT&CK categories**: Initial Access, Execution, Persistence, etc.
4. **Supply chain attacks**: typosquatting, dependency confusion, image poisoning
5. **Cloud metadata API** (169.254.169.254) is a major attack vector
6. **Persistence methods**: DaemonSets, CronJobs, webhooks
7. **DoS mitigation**: ResourceQuotas, LimitRanges, Network Policies

---

## Practice Questions

1. What is the IP address of the cloud metadata API that should be blocked?
   - Answer: 169.254.169.254

2. Which pod configuration gives full access to the host system?
   - Answer: `privileged: true`

3. What type of attack involves similar package names to trick developers?
   - Answer: Typosquatting

4. Which CNCF project is used for runtime threat detection?
   - Answer: Falco

5. What Kubernetes resource can be used for persistence via scheduled tasks?
   - Answer: CronJob

6. What framework categorizes adversary tactics and techniques?
   - Answer: MITRE ATT&CK

7. How can an attacker escape from a container to the host?
   - Answer: hostPath volumes, privileged mode, capabilities (CAP_SYS_ADMIN)

8. What prevents resource exhaustion DoS attacks?
   - Answer: ResourceQuotas and LimitRanges

---

## Additional Resources

- [MITRE ATT&CK for Containers](https://attack.mitre.org/matrices/enterprise/containers/)
- [Microsoft Threat Matrix for Kubernetes](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/)
- [Bad Pods - Kubernetes Pod Privilege Escalation](https://bishopfox.com/blog/kubernetes-pod-privilege-escalation)
- [CISA Kubernetes Hardening Guide](https://www.cisa.gov/news-events/alerts/2022/03/15/updated-kubernetes-hardening-guide)