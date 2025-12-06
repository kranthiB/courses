# KCNA Edge Cases, Advanced Scenarios & Deep Dives

## Edge Case 1: Kubernetes Object Scope Confusion

### Question Pattern
"Which of these is a NAMESPACED resource?"

**Namespaced Resources**:
- Pods, Deployments, StatefulSets, DaemonSets, ReplicaSets
- Services, Endpoints, Ingress
- ConfigMaps, Secrets
- PersistentVolumeClaims (NOT PersistentVolumes!)
- Roles, RoleBindings
- ServiceAccounts
- NetworkPolicies
- ResourceQuotas, LimitRanges

**Cluster-Scoped Resources**:
- Nodes
- Namespaces
- PersistentVolumes (NOT PVCs!)
- ClusterRoles, ClusterRoleBindings
- StorageClasses
- IngressClasses
- CustomResourceDefinitions (CRDs)
- PriorityClasses

**Memory Trick**: PV is cluster-wide (it's actual storage), PVC is namespaced (it's a request).

```bash
# Check if resource is namespaced
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

---

## Edge Case 2: Service Types Details

| Service Type | External Access | Load Balancing | Use Case |
|--------------|-----------------|----------------|----------|
| ClusterIP | No (internal only) | Yes (internal) | Internal services |
| NodePort | Yes (via node IP:port) | Yes | Development, simple external |
| LoadBalancer | Yes (via cloud LB) | Yes | Production cloud |
| ExternalName | No (DNS alias) | No | External service alias |
| Headless (ClusterIP: None) | No | No | StatefulSet DNS discovery |

**Tricky Question**: "Which service type does NOT create a ClusterIP?"
- Answer: **ExternalName** (it just creates a DNS CNAME record)

**Tricky Question**: "What happens if you create a LoadBalancer in bare-metal?"
- Answer: Stays in **Pending** state unless you have MetalLB or similar

---

## Edge Case 3: Deployment Strategies Fine Print

### Rolling Update (Default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # Can have 125% of desired during update
    maxUnavailable: 25%   # Can have 75% minimum during update
```

**Tricky**: What if desired replicas = 4?
- maxSurge: 25% = 1 pod (rounds up)
- maxUnavailable: 25% = 1 pod (rounds down)
- During update: min 3, max 5 pods

### Recreate Strategy
```yaml
strategy:
  type: Recreate
```
- **ALL pods killed** before new ones created
- Results in **downtime**
- Use for: incompatible versions, shared volumes

---

## Edge Case 4: Pod Lifecycle Phases

| Phase | Description |
|-------|-------------|
| **Pending** | Accepted but not running (scheduling, image pull) |
| **Running** | At least one container running |
| **Succeeded** | All containers terminated successfully (exit 0) |
| **Failed** | All containers terminated, at least one failed |
| **Unknown** | State cannot be determined |

**Tricky**: A pod can be in **Running** phase but NOT **Ready**!
- Running = container started
- Ready = passed readiness probe

---

## Edge Case 5: Init Containers vs Sidecar

| Aspect | Init Container | Sidecar (Regular Container) |
|--------|---------------|----------------------------|
| When runs | Before app containers | Alongside app containers |
| Restarts | Retried until success | Subject to restart policy |
| Order | Sequential | Parallel |
| Probes | No probes | Has probes |
| Use case | Setup, wait for dependency | Logging, proxy, monitoring |

**Tricky**: Init containers run **in order**, one at a time.

---

## Edge Case 6: Resource Requests vs Limits

| Aspect | Request | Limit |
|--------|---------|-------|
| Scheduling | Used for scheduling decisions | Not used for scheduling |
| Guaranteed | Reserved for the container | Maximum allowed |
| OOM Kill | Pod with lower request killed first | Pod exceeding limit killed |
| QoS Class | Affects QoS class | Affects QoS class |

**QoS Classes**:
1. **Guaranteed**: requests = limits (for ALL containers)
2. **Burstable**: At least one request or limit set, but not equal
3. **BestEffort**: No requests or limits set

**Eviction Order** (first evicted):
BestEffort → Burstable → Guaranteed

---

## Edge Case 7: ConfigMap and Secret Size Limits

- ConfigMap: **1 MiB** (1,048,576 bytes)
- Secret: **1 MiB** (same)
- etcd max value size: 1.5 MiB (but don't push it)

**Tricky**: What if you need more?
- Split into multiple ConfigMaps/Secrets
- Use external storage (Vault, S3)

---

## Edge Case 8: DNS Resolution Details

**Pod DNS Policy Options**:
| Policy | Description |
|--------|-------------|
| ClusterFirst | Use cluster DNS, fallback to upstream (DEFAULT) |
| ClusterFirstWithHostNet | Use cluster DNS even with hostNetwork |
| Default | Use node's DNS resolution |
| None | No DNS config; use dnsConfig |

**Service DNS Format**:
```
<service>.<namespace>.svc.cluster.local
```

**Pod DNS Format** (when hostname/subdomain set):
```
<hostname>.<subdomain>.<namespace>.svc.cluster.local
```

---

## DEEP DIVE SCENARIOS

## Scenario 1: Secure CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURE CI/CD PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. CODE                                                        │
│     ├─ Signed commits (GPG)                                     │
│     ├─ Branch protection                                        │
│     └─ Code review required                                     │
│                                                                 │
│  2. BUILD                                                       │
│     ├─ SAST scanning (Semgrep, SonarQube)                       │
│     ├─ SCA scanning (Snyk, Dependabot)                          │
│     └─ Minimal base images (distroless)                         │
│                                                                 │
│  3. PACKAGE                                                     │
│     ├─ Vulnerability scan (Trivy)                               │
│     ├─ SBOM generation (Syft)                                   │
│     ├─ Image signing (Cosign)                                   │
│     └─ Push to private registry                                 │
│                                                                 │
│  4. DEPLOY                                                      │
│     ├─ Signature verification (Kyverno/OPA)                     │
│     ├─ Policy enforcement                                       │
│     └─ GitOps (ArgoCD, Flux)                                    │
│                                                                 │
│  5. RUNTIME                                                     │
│     ├─ Runtime protection (Falco)                               │
│     ├─ Continuous scanning                                      │
│     └─ Audit logging                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## NUMERICAL VALUES TO MEMORIZE

### Ports

| Component | Port | Protocol |
|-----------|------|----------|
| API Server | 6443 | HTTPS |
| kubelet | 10250 | HTTPS |
| kubelet (read-only) | 10255 | HTTP |
| etcd client | 2379 | HTTPS |
| etcd peer | 2380 | HTTPS |
| kube-scheduler | 10259 | HTTPS |
| controller-manager | 10257 | HTTPS |

---