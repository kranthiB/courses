# CKS Quick Reference Card (Print-Ready)

## Exam Setup (First 2 minutes)

```bash
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
source <(kubectl completion bash)
complete -F __start_kubectl k
```

---

## 1. Network Policies

### Default Deny All
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

### Allow Specific Ingress
```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

### Block Metadata (169.254.169.254)
```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 169.254.169.254/32
```

---

## 2. RBAC

### Create Role (Imperative)
```bash
k create role NAME --verb=get,list --resource=pods -n NS
k create rolebinding NAME --role=ROLE --user=USER -n NS
k create clusterrole NAME --verb=get,list --resource=nodes
k create clusterrolebinding NAME --clusterrole=CR --user=USER
```

### Check Permissions
```bash
k auth can-i VERB RESOURCE --as USER -n NS
k auth can-i --list --as USER
k auth can-i --list --as system:serviceaccount:NS:SA
```

### ServiceAccount (No Auto-mount)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-name
automountServiceAccountToken: false
```

---

## 3. Pod Security

### PSA Namespace Labels
```bash
k label ns NAME \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

### Complete Security Context
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
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## 4. Secrets

### Create Secrets
```bash
k create secret generic NAME --from-literal=key=value
k create secret tls NAME --cert=cert.pem --key=key.pem
```

### Mount as Volume
```yaml
volumes:
- name: secret-vol
  secret:
    secretName: NAME
    defaultMode: 0400
```

### Decode Secret
```bash
k get secret NAME -o jsonpath='{.data.KEY}' | base64 -d
```

### Encryption at Rest
```yaml
# /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: [secrets]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: BASE64_KEY
  - identity: {}
```

---

## 5. AppArmor

### Profile Template
```
#include <tunables/global>
profile k8s-profile flags=(attach_disconnected) {
  #include <abstractions/base>
  file,
  deny /** w,
}
```

### Commands
```bash
aa-status                              # Check status
apparmor_parser -r /path/profile       # Load (enforce)
apparmor_parser -R /path/profile       # Remove
```

### Pod Annotation
```yaml
annotations:
  container.apparmor.security.beta.kubernetes.io/CONTAINER: localhost/PROFILE
```

---

## 6. Seccomp

### RuntimeDefault
```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

### Custom Profile
```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: profiles/custom.json
```

Profile location: `/var/lib/kubelet/seccomp/profiles/`

---

## 7. Trivy

```bash
trivy image IMAGE                           # Full scan
trivy image --severity CRITICAL IMAGE       # Filter severity
trivy image --severity CRITICAL,HIGH IMAGE  # Multiple
trivy image IMAGE | grep CVE-XXXX           # Find specific CVE
```

---

## 8. Falco

### Key Paths
```
/etc/falco/falco.yaml              # Main config
/etc/falco/falco_rules.yaml        # Default rules
/etc/falco/falco_rules.local.yaml  # Custom (override)
/etc/falco/rules.d/                # Additional rules
```

### Commands
```bash
systemctl status falco
systemctl restart falco
journalctl -fu falco
cat /var/log/syslog | grep falco
```

### Rule Template
```yaml
- rule: Rule Name
  desc: Description
  condition: spawned_process and container and proc.name = X
  output: "Alert (user=%user.name container=%container.id image=%container.image.repository)"
  priority: WARNING
  tags: [container]
```

### Common Output Fields
```
%evt.time, %user.name, %user.uid
%container.id, %container.name, %container.image.repository
%proc.name, %proc.cmdline, %proc.pname
%fd.name
```

---

## 9. Audit Logging

### Policy Template
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
- level: Metadata
  omitStages: [RequestReceived]
```

### Levels: None, Metadata, Request, RequestResponse

### Analyze Logs
```bash
cat audit.log | jq 'select(.objectRef.resource=="secrets")'
cat audit.log | jq 'select(.user.username=="USER")'
cat audit.log | jq 'select(.verb=="delete")'
cat audit.log | jq 'select(.responseStatus.code >= 400)'
```

---

## 10. Certificates

```bash
# Generate key + CSR
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=user/O=group"

# Create K8s CSR
cat <<EOF | k apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user-csr
spec:
  request: $(cat user.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages: [client auth]
EOF

# Approve + Get
k certificate approve user-csr
k get csr user-csr -o jsonpath='{.status.certificate}' | base64 -d > user.crt
```

---

## 11. RuntimeClass

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: gvisor
```

---

## 12. Ingress TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts: [example.com]
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc
            port:
              number: 80
```

---

## Quick Fixes

### Pod Won't Start (Security Context)
Add required emptyDir volumes for read-only filesystem:
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

### API Server Not Starting
1. Check logs: `crictl logs $(crictl ps -a | grep apiserver | awk '{print $1}')`
2. Check manifest: `/etc/kubernetes/manifests/kube-apiserver.yaml`
3. Fix YAML errors
4. Wait for restart (don't manually restart)

### Permission Denied
```bash
k auth can-i VERB RESOURCE --as USER -n NS
# Check if RBAC exists and is bound correctly
```

---

## Exam Strategy

| Time | Action |
|------|--------|
| 0-5 | Setup aliases, scan questions |
| 5-60 | Easy/medium questions |
| 60-100 | Harder questions |
| 100-115 | Flagged questions |
| 115-120 | Verify answers |

**Rules**:
- Max 8 min per question
- Flag and move on if stuck
- Always verify solutions work
- Watch namespaces!

---

**Good luck! You're prepared for this.**