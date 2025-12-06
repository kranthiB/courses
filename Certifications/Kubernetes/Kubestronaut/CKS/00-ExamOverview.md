# CKS (Certified Kubernetes Security Specialist) - Complete Study Guide

## Exam Overview

| Attribute | Details |
|-----------|---------|
| **Exam Name** | Certified Kubernetes Security Specialist (CKS) |
| **Duration** | 2 hours |
| **Questions** | 15-20 performance-based tasks |
| **Passing Score** | 67% |
| **Kubernetes Version** | 1.31 (as of 2024) |
| **Prerequisite** | Valid CKA certification |
| **Validity** | 2 years |
| **Cost** | $445 USD (includes one free retake) |
| **Format** | Online, proctored, performance-based |

---

## Domain Weight Distribution

| Domain | Weight | Study Guide File |
|--------|--------|------------------|
| Cluster Setup | 10% | 01-Domain1-ClusterSetup.md |
| Cluster Hardening | 15% | 02-Domain2-ClusterHardening.md |
| System Hardening | 15% | 03-Domain3-SystemHardening.md |
| Minimize Microservice Vulnerabilities | 20% | 04-Domain4-MinimizeMicroserviceVulnerabilities.md |
| Supply Chain Security | 20% | 05-Domain5-SupplyChainSecurity.md |
| Monitoring, Logging & Runtime Security | 20% | 06-Domain6-MonitoringLoggingandRuntimeSecurity.md |

---

## Allowed Resources During Exam

You can access these sites during the exam:

- https://kubernetes.io/docs/
- https://kubernetes.io/blog/
- https://github.com/kubernetes/
- https://falco.org/docs/
- https://aquasecurity.github.io/trivy/
- https://gitlab.com/apparmor/apparmor/-/wikis/Documentation

**Tip**: Bookmark important pages before the exam!

---

## Essential Tools & Commands Cheat Sheet

### kubectl Basics

```bash
# Create resources
kubectl apply -f manifest.yaml
kubectl create -f manifest.yaml

# Get resources
kubectl get pods -A
kubectl get pods -o wide
kubectl get pods -o yaml

# Describe resources
kubectl describe pod <pod-name>

# Delete resources
kubectl delete pod <pod-name>
kubectl delete -f manifest.yaml

# Execute commands in pods
kubectl exec -it <pod> -- /bin/sh

# Port forwarding
kubectl port-forward <pod> 8080:80

# Check permissions
kubectl auth can-i <verb> <resource>
kubectl auth can-i --list
```

### RBAC Quick Commands

```bash
# Create role
kubectl create role <name> --verb=get,list --resource=pods -n <namespace>

# Create rolebinding
kubectl create rolebinding <name> --role=<role> --user=<user> -n <namespace>

# Create clusterrole
kubectl create clusterrole <name> --verb=get,list --resource=pods

# Create clusterrolebinding
kubectl create clusterrolebinding <name> --clusterrole=<role> --user=<user>

# Check permissions as another user
kubectl auth can-i create pods --as <user>
kubectl auth can-i --list --as system:serviceaccount:<ns>:<sa>
```

### Secrets

```bash
# Create secret
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret tls <name> --cert=cert.pem --key=key.pem

# View secret (encoded)
kubectl get secret <name> -o yaml

# Decode secret
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d
```

### Network Policy

```bash
# Get network policies
kubectl get networkpolicies -A

# Describe network policy
kubectl describe networkpolicy <name> -n <namespace>
```

### Security Tools

```bash
# Trivy - Image scanning
trivy image <image-name>
trivy image --severity HIGH,CRITICAL <image-name>

# Kubesec - Static analysis
kubesec scan pod.yaml

# Falco
systemctl status falco
journalctl -fu falco
cat /var/log/syslog | grep falco

# AppArmor
aa-status
apparmor_parser -r /etc/apparmor.d/<profile>
apparmor_parser -R /etc/apparmor.d/<profile>

# Seccomp profiles
/var/lib/kubelet/seccomp/profiles/
```

### Certificate Management

```bash
# Generate private key
openssl genrsa -out user.key 2048

# Generate CSR
openssl req -new -key user.key -out user.csr -subj "/CN=user/O=group"

# Create CertificateSigningRequest
kubectl apply -f csr.yaml

# Approve CSR
kubectl certificate approve <csr-name>

# Get certificate
kubectl get csr <csr-name> -o jsonpath='{.status.certificate}' | base64 -d > user.crt
```

---

## Critical Exam Topics

### 1. Network Policies (MUST KNOW)

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress

# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
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
    ports:
    - port: 8080
```

### 2. Pod Security Standards (MUST KNOW)

```yaml
# Namespace with PSS enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 3. Security Context (MUST KNOW)

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

### 4. ServiceAccount Security (MUST KNOW)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
automountServiceAccountToken: false
```

### 5. Falco Rules (MUST KNOW)

```yaml
- rule: <name>
  desc: <description>
  condition: <filter>
  output: <format>
  priority: <level>
```

### 6. Audit Policy (MUST KNOW)

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
```

---

## Exam Strategy

### Time Management

| Phase | Time | Activities |
|-------|------|------------|
| Quick wins | 0-40 min | Easy questions (1-3 min each) |
| Medium | 40-90 min | Standard questions (5-8 min each) |
| Hard | 90-110 min | Complex questions (10-15 min each) |
| Review | 110-120 min | Review flagged questions |

### Question Approach

1. **Read carefully** - Understand what's being asked
2. **Check the context** - Note namespace, cluster
3. **Plan your approach** - Think before typing
4. **Use documentation** - Don't memorize everything
5. **Verify your work** - Test your solution
6. **Flag and move on** - Don't get stuck

### Common Mistakes to Avoid

- Forgetting to switch namespace (`-n <namespace>`)
- Not waiting for API server restart after changes
- Typos in YAML manifests
- Missing required volumes for read-only filesystems
- Not verifying your solutions work
- Spending too much time on one question

---

## Pre-Exam Checklist

### Environment Setup
- [ ] Stable internet connection
- [ ] Quiet room
- [ ] Valid ID ready
- [ ] System requirements met
- [ ] PSI browser installed and tested

### Knowledge Check
- [ ] Network Policies
- [ ] RBAC (Roles, RoleBindings)
- [ ] Pod Security Standards
- [ ] Security Contexts
- [ ] Secrets management
- [ ] AppArmor profiles
- [ ] Seccomp profiles
- [ ] Falco rules
- [ ] Audit policies
- [ ] Trivy scanning
- [ ] Image security

### Practice
- [ ] Completed killer.sh simulator
- [ ] Practiced on multiple clusters
- [ ] Comfortable with kubectl
- [ ] Know documentation structure

---

## Quick Reference Links

### Kubernetes Documentation
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
- [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
- [Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)

### External Tools (for study)
- [Falco](https://falco.org/docs/)
- [Trivy](https://aquasecurity.github.io/trivy/)
- [Kyverno](https://kyverno.io/docs/)
- [OPA/Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)

---

## Final Tips

1. **Practice, practice, practice** - Use killer.sh, KodeKloud, and real clusters
2. **Speed matters** - Know commands by heart
3. **Use aliases** - `alias k=kubectl`
4. **Know the docs** - Navigate quickly
5. **Stay calm** - 2 hours is enough if prepared
6. **Read questions twice** - Understand before acting
7. **Test your work** - Verify solutions work

**Good luck on your CKS exam!**