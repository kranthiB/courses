# KCSA Exam Study Guide: Domain 3 - Kubernetes Security Fundamentals

## Domain Weight: 22% (~13 questions out of 60)

This domain covers core security mechanisms: Pod Security Standards, Authentication, Authorization (RBAC), Secrets management, and Network Policies.

---

## 3.1 Pod Security Standards (PSS)

### Overview

Pod Security Standards define three security levels for pods, replacing the deprecated PodSecurityPolicy (PSP).

```
┌───────────────────────────────────────────────────────────────────┐
│                    Pod Security Standards                         │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PRIVILEGED ──────────────────────────────────────────► MOST      │
│  • Unrestricted                                         PERMISSIVE│
│  • Full access                                                    │
│  • Use for: System workloads, trusted admins                      │
│                                                                   │
│  BASELINE ─────────────────────────────────────────────►          │
│  • Minimally restrictive                                          │
│  • Prevents known privilege escalations                           │
│  • Use for: Most workloads                                        │
│                                                                   │
│  RESTRICTED ───────────────────────────────────────────► MOST     │
│  • Heavily restricted                                   SECURE    │
│  • Best practices enforcement                                     │
│  • Use for: Security-sensitive workloads                          │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Pod Security Standards Comparison

| Restriction | Privileged | Baseline | Restricted |
|-------------|------------|----------|------------|
| Host namespaces (hostNetwork, hostPID, hostIPC) | Allowed | **Blocked** | **Blocked** |
| Privileged containers | Allowed | **Blocked** | **Blocked** |
| Linux capabilities | Any | Drop dangerous | Drop ALL |
| HostPath volumes | Allowed | Allowed | **Blocked** |
| Host ports | Allowed | Limited | **Blocked** |
| AppArmor | Any | Any | RuntimeDefault |
| Seccomp | Any | Any | RuntimeDefault/Localhost |
| Run as non-root | Not required | Not required | **Required** |
| RunAsUser | Any | Any | Non-root ranges |
| Privilege escalation | Allowed | Allowed | **Blocked** |
| Root filesystem | Read-write | Read-write | Read-only recommended |

### Pod Security Admission (PSA)

PSA is the built-in admission controller that enforces Pod Security Standards.

**Enforcement Modes**:
| Mode | Behavior |
|------|----------|
| `enforce` | Violations **reject** the pod |
| `audit` | Violations logged in audit log, pod allowed |
| `warn` | Violations show warning to user, pod allowed |

**Namespace Labels for PSA**:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted policy
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    
    # Audit and warn with baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
```

**Example: Applying PSA to Existing Namespace**:
```bash
# Apply restricted enforcement
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# Check namespace labels
kubectl get namespace production --show-labels
```

### Pod Security Policy (PSP) - DEPRECATED

**Important**: PSP was removed in Kubernetes 1.25. Know this for the exam!

- PSP = Deprecated/Removed in K8s 1.25
- Pod Security Admission = Replacement (built-in)
- Third-party alternatives: OPA/Gatekeeper, Kyverno

---

## 3.2 Authentication

### Authentication Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────┐
│   Request   │───►│   AuthN     │───►│   AuthZ     │───►│ Admission│
│             │    │(Who are you)│    │(What can you│    │ Control  │
│             │    │             │    │    do?)     │    │          │
└─────────────┘    └─────────────┘    └─────────────┘    └──────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    Certificates    Tokens         OIDC/Webhook
```

### Authentication Methods

| Method | Description | Use Case |
|--------|-------------|----------|
| **X.509 Client Certificates** | PKI-based authentication | Admins, kubeadm setup |
| **Bearer Tokens** | Static or service account tokens | Service accounts |
| **Bootstrap Tokens** | Temporary tokens for joining nodes | Cluster bootstrap |
| **OIDC (OpenID Connect)** | External identity provider | Enterprise SSO |
| **Webhook Token Auth** | External authentication service | Custom auth |
| **Authenticating Proxy** | Proxy handles auth | API gateways |

### Service Account Authentication

**Service Account Auto-mounting**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  automountServiceAccountToken: false  # Disable if not needed!
```

**Service Account Token Volume Projection** (Recommended):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600  # 1 hour expiry
          audience: api
```

### Authentication Best Practices

1. **Disable anonymous authentication** in production
2. **Use OIDC** for human users (not certificates)
3. **Rotate credentials** regularly
4. **Use short-lived tokens** for service accounts
5. **Don't share service accounts** between workloads
6. **Audit authentication events**

---

## 3.3 Authorization (RBAC)

### RBAC Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         RBAC Model                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   WHO (Subjects)          WHAT (Resources)      HOW (Verbs)     │
│   ├─ Users                ├─ pods               ├─ get          │
│   ├─ Groups               ├─ deployments        ├─ list         │
│   └─ ServiceAccounts      ├─ services           ├─ watch        │
│                           ├─ secrets            ├─ create       │
│                           ├─ configmaps         ├─ update       │
│                           └─ ...                ├─ patch        │
│                                                 ├─ delete       │
│                                                 └─ ...          │
│                                                                 │
│   Role/ClusterRole: Defines permissions (verbs on resources)    │
│   RoleBinding/ClusterRoleBinding: Assigns roles to subjects     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Role vs ClusterRole

| Type | Scope | Use Case |
|------|-------|----------|
| **Role** | Namespace-scoped | Per-namespace permissions |
| **ClusterRole** | Cluster-wide | Cross-namespace or cluster resources |
| **RoleBinding** | Namespace-scoped | Bind Role/ClusterRole in namespace |
| **ClusterRoleBinding** | Cluster-wide | Bind ClusterRole cluster-wide |

### RBAC Examples

**Role (Namespace-scoped)**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

**RoleBinding**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: app-sa
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRole**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

**ClusterRoleBinding**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: security-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Best Practices

1. **Least Privilege**: Grant minimum required permissions
2. **Avoid wildcards**: Don't use `*` for resources or verbs
3. **Don't use cluster-admin**: Except for break-glass scenarios
4. **Namespace isolation**: Use Roles over ClusterRoles
5. **Regular audits**: Review RBAC permissions periodically
6. **Avoid system:masters**: Members bypass ALL RBAC checks

### RBAC Verification

```bash
# Check if user can perform action
kubectl auth can-i create pods --as=jane
kubectl auth can-i delete secrets --as=system:serviceaccount:default:my-sa

# List all roles in namespace
kubectl get roles -n development

# Describe role permissions
kubectl describe role pod-reader -n development

# Check who has access to what
kubectl auth can-i --list --as=jane
```

---

## 3.4 Secrets Management

### Kubernetes Secrets

**Critical**: Secrets are base64 ENCODED, not ENCRYPTED by default!

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=        # base64 encoded "admin"
  password: cGFzc3dvcmQ=    # base64 encoded "password"
```

**Decode a secret**:
```bash
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Generic secret (default) |
| `kubernetes.io/service-account-token` | Service account token |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH private key |

### Encrypting Secrets at Rest

**Encryption Configuration**:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}   # Fallback to unencrypted (for reading old secrets)
```

**Enable on API Server**:
```yaml
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### Using Secrets Securely

**As Environment Variables** (Less Secure):
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: my-secret
      key: password
```

**As Volume Mount** (More Secure):
```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true
volumes:
- name: secret-volume
  secret:
    secretName: my-secret
```

### External Secret Management

| Solution | Description |
|----------|-------------|
| **HashiCorp Vault** | Enterprise secrets management |
| **AWS Secrets Manager** | AWS-native secrets |
| **Azure Key Vault** | Azure-native secrets |
| **GCP Secret Manager** | GCP-native secrets |
| **External Secrets Operator** | Sync external secrets to K8s |
| **Sealed Secrets** | Encrypt secrets for GitOps |

### Secrets Best Practices

1. **Enable encryption at rest**
2. **Use RBAC** to restrict secret access
3. **Avoid environment variables** for sensitive data
4. **Don't log secrets**
5. **Rotate secrets** regularly
6. **Use external secret managers** for production
7. **Never commit secrets** to Git

---

## 3.5 Isolation and Segmentation

### Multi-tenancy Models

| Model | Isolation Level | Description |
|-------|-----------------|-------------|
| **Namespaces** | Soft | Logical separation, shared cluster |
| **Network Policies** | Network | Traffic isolation |
| **Resource Quotas** | Resources | CPU/memory limits per namespace |
| **RBAC** | Access | Permission boundaries |
| **Separate Clusters** | Hard | Complete isolation |

### Namespace Isolation

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    tenant: a
---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
---
# Limit Range
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-a-limits
  namespace: tenant-a
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

### Node Isolation

**Node Selectors**:
```yaml
spec:
  nodeSelector:
    security-zone: high
```

**Taints and Tolerations**:
```bash
# Taint a node
kubectl taint nodes node1 security=high:NoSchedule
```

```yaml
# Pod with toleration
spec:
  tolerations:
  - key: "security"
    operator: "Equal"
    value: "high"
    effect: "NoSchedule"
```

---

## 3.6 Audit Logging

### Audit Policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all requests to secrets at Metadata level
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]

# Log pod changes at RequestResponse level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["create", "update", "patch", "delete"]

# Don't log read-only requests to configmaps
- level: None
  resources:
  - group: ""
    resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

# Log everything else at Metadata level
- level: Metadata
```

### Audit Levels

| Level | What's Logged |
|-------|---------------|
| `None` | Nothing |
| `Metadata` | Request metadata (user, timestamp, resource) |
| `Request` | Metadata + request body |
| `RequestResponse` | Metadata + request + response bodies |

---

## Key Exam Tips for This Domain

1. **PSP is REMOVED** in K8s 1.25 - replaced by Pod Security Admission
2. **Three PSS levels**: Privileged > Baseline > Restricted
3. **Secrets are base64 encoded**, NOT encrypted by default
4. **RBAC**: Role (namespace) vs ClusterRole (cluster-wide)
5. **Least privilege** is the key RBAC principle
6. **automountServiceAccountToken: false** when token not needed
7. **Audit logging** captures who did what

---

## Practice Questions

1. What replaced PodSecurityPolicy in Kubernetes 1.25?
   - Answer: Pod Security Admission (PSA)

2. What are the three Pod Security Standards levels?
   - Answer: Privileged, Baseline, Restricted

3. Are Kubernetes Secrets encrypted by default?
   - Answer: No, they are base64 encoded (not encrypted)

4. What is the difference between Role and ClusterRole?
   - Answer: Role is namespace-scoped; ClusterRole is cluster-wide

5. Which RBAC resource assigns a Role to a user?
   - Answer: RoleBinding

6. What flag enables encryption of secrets at rest?
   - Answer: `--encryption-provider-config`

7. Which Pod Security Admission mode rejects non-compliant pods?
   - Answer: enforce

8. What verbs are typically used for read-only access?
   - Answer: get, list, watch

---

## Additional Resources

- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Managing Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)