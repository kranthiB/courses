# CKS Speed Drills & Command Reference

## Purpose

These drills build muscle memory for commands you'll need during the exam. Practice each section until you can complete it without looking at the reference.

**Goal**: Complete each drill set in under 2 minutes

---

## Essential Aliases (Add to ~/.bashrc)

```bash
# Add these FIRST in every exam session
alias k=kubectl
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias ka='kubectl apply -f'
alias kx='kubectl exec -it'
alias kl='kubectl logs'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kns='kubectl config set-context --current --namespace'

# Auto-completion
source <(kubectl completion bash)
complete -F __start_kubectl k

# Quick export
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
```

---

## Drill Set 1: Pod Creation Speed (Target: 2 minutes)

### Exercise 1.1: Create pods with security context

```bash
# Create pod running as user 1000
k run secure1 --image=nginx $do | sed '/containers:/a\    securityContext:\n      runAsUser: 1000\n      runAsNonRoot: true' | k apply -f -

# FASTER method - memorize this template:
k apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure1
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
  containers:
  - name: main
    image: nginx
EOF
```

### Exercise 1.2: Quick pod with all security hardening

Practice typing this from memory:

```bash
k apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hardened
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
EOF
```

**Drill**: Type this 5 times without looking. Time yourself.

---

## Drill Set 2: RBAC Speed (Target: 3 minutes)

### Exercise 2.1: Create Role and RoleBinding

```bash
# Create role (imperative)
k create role pod-reader --verb=get,list,watch --resource=pods -n dev

# Create rolebinding (imperative)  
k create rolebinding pod-reader-binding --role=pod-reader --user=jane -n dev

# Verify
k auth can-i list pods -n dev --as jane
```

### Exercise 2.2: ServiceAccount with minimal permissions

```bash
# Create SA
k create sa app-sa -n default

# Create role
k create role app-role --verb=get --resource=configmaps -n default

# Bind
k create rolebinding app-binding --role=app-role --serviceaccount=default:app-sa -n default

# Verify
k auth can-i get configmaps -n default --as system:serviceaccount:default:app-sa
k auth can-i list configmaps -n default --as system:serviceaccount:default:app-sa
```

### Exercise 2.3: ClusterRole and ClusterRoleBinding

```bash
# Create clusterrole
k create clusterrole node-reader --verb=get,list,watch --resource=nodes

# Create clusterrolebinding
k create clusterrolebinding node-reader-binding --clusterrole=node-reader --group=developers

# Verify
k auth can-i list nodes --as-group=developers --as=test
```

**Drill**: Create these RBAC resources for 3 different scenarios without looking.

---

## Drill Set 3: Network Policy Speed (Target: 2 minutes)

### Exercise 3.1: Default deny all

```bash
k apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: secure
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

### Exercise 3.2: Allow specific ingress

```bash
k apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: app
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
EOF
```

### Exercise 3.3: Block metadata API

```bash
k apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
EOF
```

**Drill**: Write all three policies from memory in under 3 minutes.

---

## Drill Set 4: Secrets Speed (Target: 2 minutes)

### Exercise 4.1: Create secrets

```bash
# From literal
k create secret generic db-creds --from-literal=user=admin --from-literal=pass=secret123

# From file
k create secret generic ssh-key --from-file=id_rsa=/path/to/key

# TLS
k create secret tls web-tls --cert=cert.pem --key=key.pem

# Docker registry
k create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass
```

### Exercise 4.2: Mount secret as volume

```bash
k apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-creds
      defaultMode: 0400
EOF
```

### Exercise 4.3: View/decode secrets

```bash
# View secret
k get secret db-creds -o yaml

# Decode specific key
k get secret db-creds -o jsonpath='{.data.user}' | base64 -d

# Decode all
k get secret db-creds -o json | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
```

---

## Drill Set 5: Security Context Speed (Target: 2 minutes)

### Exercise 5.1: Type this pod spec from memory

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop:
        - ALL
```

### Exercise 5.2: Add volumes for read-only filesystem

```yaml
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

---

## Drill Set 6: PSA Labels Speed (Target: 1 minute)

### Exercise 6.1: Add all three modes

```bash
k label ns secure \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest
```

### Exercise 6.2: Create namespace with labels

```bash
k apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
EOF
```

---

## Drill Set 7: Trivy Commands (Target: 1 minute)

```bash
# Basic scan
trivy image nginx:1.25

# Filter severity
trivy image --severity CRITICAL,HIGH nginx:1.25

# JSON output
trivy image --format json nginx:1.25 > scan.json

# Scan with exit code (for CI)
trivy image --exit-code 1 --severity CRITICAL nginx:1.25

# Scan specific CVE
trivy image nginx:1.25 | grep CVE-2023-1234

# Count vulnerabilities
trivy image nginx:1.25 2>/dev/null | grep -c CRITICAL
```

---

## Drill Set 8: Falco Commands (Target: 2 minutes)

```bash
# Check status
systemctl status falco

# View logs
journalctl -fu falco
cat /var/log/syslog | grep falco

# Restart after config change
systemctl restart falco

# Validate rules
falco --validate /etc/falco/falco_rules.yaml

# List available fields
falco --list

# Key file locations
/etc/falco/falco.yaml              # Main config
/etc/falco/falco_rules.yaml        # Default rules
/etc/falco/falco_rules.local.yaml  # Custom rules (override)
/etc/falco/rules.d/                # Additional rules
```

### Falco Rule Template (Memorize!)

```yaml
- rule: Rule Name Here
  desc: Description of rule
  condition: spawned_process and container and proc.name = suspicious
  output: "Alert message (user=%user.name container=%container.id image=%container.image.repository)"
  priority: WARNING
  tags: [container, security]
```

---

## Drill Set 9: AppArmor Commands (Target: 2 minutes)

```bash
# Check status
aa-status

# Load profile (enforce)
apparmor_parser -r /etc/apparmor.d/profile

# Load profile (complain)
apparmor_parser -C /etc/apparmor.d/profile

# Remove profile
apparmor_parser -R /etc/apparmor.d/profile

# Pod annotation format
container.apparmor.security.beta.kubernetes.io/<container-name>: localhost/<profile-name>
```

### AppArmor Profile Template

```bash
#include <tunables/global>

profile k8s-custom flags=(attach_disconnected) {
  #include <abstractions/base>
  file,
  deny /** w,
}
```

---

## Drill Set 10: Audit Log Analysis (Target: 2 minutes)

```bash
# Find all secret access
cat audit.log | jq 'select(.objectRef.resource=="secrets")'

# Find specific user
cat audit.log | jq 'select(.user.username=="baduser")'

# Find failed requests
cat audit.log | jq 'select(.responseStatus.code >= 400)'

# Find delete operations
cat audit.log | jq 'select(.verb=="delete")'

# Find namespace-specific
cat audit.log | jq 'select(.objectRef.namespace=="kube-system")'

# Count by user
cat audit.log | jq -r '.user.username' | sort | uniq -c | sort -rn
```

---

## Drill Set 11: Certificate Commands (Target: 2 minutes)

```bash
# Generate key
openssl genrsa -out user.key 2048

# Generate CSR
openssl req -new -key user.key -out user.csr -subj "/CN=user/O=group"

# View CSR
openssl req -in user.csr -text -noout

# Create CSR in K8s
cat <<EOF | k apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user-csr
spec:
  request: $(cat user.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# Approve
k certificate approve user-csr

# Get cert
k get csr user-csr -o jsonpath='{.status.certificate}' | base64 -d > user.crt
```

---

## Drill Set 12: Quick Verification Commands

```bash
# Check RBAC
k auth can-i <verb> <resource> --as <user> -n <namespace>
k auth can-i --list --as <user>

# Check pod security context
k get pod <name> -o jsonpath='{.spec.securityContext}'
k get pod <name> -o jsonpath='{.spec.containers[*].securityContext}'

# Check if token mounted
k exec <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount/

# Test network policy
k exec <pod> -- curl -s --max-time 3 <target>
k exec <pod> -- nc -zv <ip> <port>

# Check AppArmor
k get pod <name> -o jsonpath='{.metadata.annotations}'

# Check seccomp
k get pod <name> -o jsonpath='{.spec.securityContext.seccompProfile}'
```

---

## Speed Test: Full Security Pod

**Challenge**: Create this entire pod from memory in under 60 seconds:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ultra-secure
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10000
    runAsGroup: 10000
    fsGroup: 10000
    seccompProfile:
      type: RuntimeDefault
  serviceAccountName: restricted-sa
  automountServiceAccountToken: false
  containers:
  - name: app
    image: nginx:1.25-alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: "100m"
        memory: "128Mi"
      requests:
        cpu: "50m"
        memory: "64Mi"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

---

## Daily Practice Routine

### Day 1-3: Foundation
- [ ] Practice Drill Sets 1-4 (3 times each)
- [ ] Complete 5 labs from practice labs

### Day 4-6: Intermediate
- [ ] Practice Drill Sets 5-8 (3 times each)
- [ ] Complete 10 labs from practice labs
- [ ] Time yourself on drills

### Day 7-9: Advanced
- [ ] Practice Drill Sets 9-12 (3 times each)
- [ ] Complete Mock Exam 1
- [ ] Review mistakes

### Day 10-12: Mastery
- [ ] All drills under target time
- [ ] Complete Mock Exam 2
- [ ] Full Speed Test challenge

### Day 13-14: Final Prep
- [ ] Killer.sh simulator attempt 1
- [ ] Review all weak areas
- [ ] Killer.sh simulator attempt 2

---

## Progress Tracker

| Drill Set | Target | Best Time | ✓ Under Target |
|-----------|--------|-----------|----------------|
| 1. Pod Creation | 2 min | | ☐ |
| 2. RBAC | 3 min | | ☐ |
| 3. NetworkPolicy | 2 min | | ☐ |
| 4. Secrets | 2 min | | ☐ |
| 5. Security Context | 2 min | | ☐ |
| 6. PSA Labels | 1 min | | ☐ |
| 7. Trivy | 1 min | | ☐ |
| 8. Falco | 2 min | | ☐ |
| 9. AppArmor | 2 min | | ☐ |
| 10. Audit Logs | 2 min | | ☐ |
| 11. Certificates | 2 min | | ☐ |
| 12. Verification | 1 min | | ☐ |
| **Full Security Pod** | 60 sec | | ☐ |