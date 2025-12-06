# KCSA Edge Cases, Advanced Scenarios & Deep Dives

## Edge Case 1: PSS vs PSA vs PSP

| Term | Full Name | Status |
|------|-----------|--------|
| **PSP** | PodSecurityPolicy | **REMOVED** in K8s 1.25 |
| **PSS** | Pod Security Standards | The **standard/rules** (Privileged, Baseline, Restricted) |
| **PSA** | Pod Security Admission | The **admission controller** that enforces PSS |

**Tricky Question**: "What replaced PodSecurityPolicy?"
- Answer: **Pod Security Admission** (NOT Pod Security Standards)
- PSS defines the levels; PSA enforces them

---

## Edge Case 2: RBAC Aggregation

ClusterRoles can aggregate other ClusterRoles:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
  labels:
    rbac.example.com/aggregate-to-admin: "true"
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

The `admin` ClusterRole aggregates roles with matching labels automatically.

**Built-in Aggregated Roles**:
- `admin` aggregates: `rbac.authorization.k8s.io/aggregate-to-admin: "true"`
- `edit` aggregates: `rbac.authorization.k8s.io/aggregate-to-edit: "true"`
- `view` aggregates: `rbac.authorization.k8s.io/aggregate-to-view: "true"`

---

## Edge Case 3: Service Account Token Details

**Old (Pre-1.24)**:
- Auto-created, non-expiring token as Secret
- Mounted automatically

**New (1.24+)**:
- TokenRequest API: short-lived, auto-rotated
- Bound to pod lifetime
- Default expiration: 1 hour (configurable)

```yaml
# Projected token volume (new method)
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: api
```

**Tricky**: Legacy tokens still work but are deprecated.

---

## Edge Case 4: Network Policy Subtleties

### Empty vs Missing Selectors

```yaml
# Empty podSelector = ALL pods in namespace
spec:
  podSelector: {}

# Missing ingress/egress = no rules = no traffic allowed
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

### CIDR Exceptions
```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.0.1.0/24  # Except this subnet
```

### AND vs OR Logic
```yaml
# OR - either pods with label OR from namespace
- from:
  - podSelector:
      matchLabels:
        app: web
  - namespaceSelector:
      matchLabels:
        env: prod

# AND - pods with label AND from namespace
- from:
  - podSelector:
      matchLabels:
        app: web
    namespaceSelector:
      matchLabels:
        env: prod
```

**Key**: Separate `-` means OR; same `-` means AND.

---

## Edge Case 5: Admission Controller Order (Detailed)

```
Request
   │
   ▼
Authentication (WHO are you?)
   │
   ▼
Authorization (CAN you do this?) - RBAC, ABAC, Webhook
   │
   ▼
Mutating Admission (MODIFY the request)
   │    ├─ MutatingAdmissionWebhook
   │    ├─ PodPreset (deprecated)
   │    └─ DefaultStorageClass
   │
   ▼
Object Schema Validation
   │
   ▼
Validating Admission (ACCEPT or REJECT)
   │    ├─ ValidatingAdmissionWebhook
   │    ├─ PodSecurity (PSA)
   │    ├─ ResourceQuota
   │    └─ LimitRanger
   │
   ▼
Persist to etcd
```

**Tricky**: Some controllers are BOTH mutating and validating (ResourceQuota).

---

## Edge Case 6: Encryption at Rest Configuration

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  # Order matters! First provider is used for encryption
  - aescbc:  # or aesgcm
      keys:
      - name: key1
        secret: <base64-encoded-key>
  - identity: {}  # Fallback for reading unencrypted data
```

**Tricky Points**:
- `identity` provider = NO encryption (plaintext)
- First provider = used for WRITING
- All providers = tried for READING (in order)
- Must restart API server after change
- Must re-encrypt existing secrets manually

```bash
# Re-encrypt all secrets after enabling encryption
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

---

## Edge Case 7: Audit Log Levels

| Level | Logged Data |
|-------|-------------|
| **None** | Nothing |
| **Metadata** | Request metadata only (user, timestamp, resource, verb) |
| **Request** | Metadata + Request body |
| **RequestResponse** | Metadata + Request body + Response body |

**When NOT to use RequestResponse**:
- Secrets (logs secret values!)
- Large responses (performance)
- High-frequency endpoints (storage)

**Tricky**: `level: RequestResponse` on secrets will log secret values in audit log!

---

## Edge Case 8: Container Security Capabilities

**Dangerous Capabilities**:
| Capability | Risk |
|------------|------|
| `CAP_SYS_ADMIN` | Almost equivalent to root |
| `CAP_NET_ADMIN` | Modify network settings |
| `CAP_NET_RAW` | Raw socket access (packet sniffing) |
| `CAP_SYS_PTRACE` | Debug other processes |
| `CAP_DAC_OVERRIDE` | Bypass file permissions |

**Required Capabilities** (sometimes needed):
| Capability | Use Case |
|------------|----------|
| `CAP_NET_BIND_SERVICE` | Bind to ports < 1024 |
| `CAP_CHOWN` | Change file ownership |
| `CAP_SETUID/SETGID` | Change user/group IDs |

**Best Practice**:
```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # Only if needed
```

---

## Edge Case 9: etcd Security Details

**etcd Ports**:
| Port | Purpose | Security |
|------|---------|----------|
| 2379 | Client communication | Requires mTLS |
| 2380 | Peer communication | Requires mTLS |

**etcd Data at Risk**:
- Secrets (readable in plaintext unless encrypted at rest)
- Service account tokens
- TLS certificates
- All cluster state

**Tricky**: If attacker gets etcd access, they have EVERYTHING.

---

## Edge Case 10: STRIDE Mitigations Matrix

| Threat | Kubernetes Mitigation |
|--------|----------------------|
| **Spoofing** | Strong authentication, mTLS, OIDC |
| **Tampering** | Image signing, admission control, RBAC |
| **Repudiation** | Audit logging, immutable logs |
| **Info Disclosure** | Encryption (rest & transit), NetworkPolicies |
| **Denial of Service** | ResourceQuotas, LimitRanges, API rate limiting |
| **Elevation** | PSA, RBAC, least privilege, no privileged pods |

---

## DEEP DIVE SCENARIOS

### Scenario 1: Multi-Tenant Cluster Security

**Requirements**:
1. Namespace isolation
2. Network segmentation
3. Resource quotas
4. Separate service accounts
5. RBAC per tenant

```yaml
# 1. Create namespace with PSA
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    pod-security.kubernetes.io/enforce: restricted

---
# 2. Default deny NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 3. ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"

---
# 4. LimitRange (defaults)
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-a-limits
  namespace: tenant-a
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

---

### Scenario 2: Incident Response

**Suspected Compromise - Response Steps**:

1. **Contain**
   ```bash
   # Isolate namespace with NetworkPolicy
   kubectl apply -f emergency-deny-all.yaml -n compromised-ns
   
   # Scale down suspicious workloads
   kubectl scale deployment suspicious-app --replicas=0 -n compromised-ns
   ```

2. **Collect Evidence**
   ```bash
   # Get audit logs
   kubectl logs -n kube-system -l component=kube-apiserver --tail=10000 > audit.log
   
   # Export pod specs
   kubectl get pods -n compromised-ns -o yaml > pods.yaml
   
   # Check for persistence mechanisms
   kubectl get cronjobs,daemonsets -A -o yaml > persistence-check.yaml
   ```

3. **Investigate**
   ```bash
   # Check RBAC
   kubectl auth can-i --list --as=system:serviceaccount:compromised-ns:default
   
   # Check network policies
   kubectl get networkpolicies -n compromised-ns
   
   # Check secrets access
   kubectl get secrets -n compromised-ns
   ```

4. **Remediate**
   - Rotate compromised credentials
   - Patch vulnerabilities
   - Update policies
   - Re-deploy clean images

5. **Document & Improve**
   - Post-mortem
   - Update security policies
   - Improve monitoring

---

## NUMERICAL VALUES TO MEMORIZE

### Limits & Defaults

| Resource | Default/Limit |
|----------|---------------|
| Secret/ConfigMap size | 1 MiB |
| Max pods per node | ~110 (default) |
| Service NodePort range | 30000-32767 |
| Default service account token expiry | 1 hour |
| Audit log recommended max size | 100 MB |
| CIS kubeconfig permissions | 600 |

## IP Addresses

| Address | Purpose |
|---------|---------|
| 169.254.169.254 | Cloud metadata service |
| 10.0.0.0/8 | Common pod CIDR |
| 172.16.0.0/12 | Common service CIDR |
| 192.168.0.0/16 | Common node CIDR |

---
