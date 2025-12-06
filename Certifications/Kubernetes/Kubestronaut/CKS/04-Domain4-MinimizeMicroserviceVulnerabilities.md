# CKS Domain 4: Minimize Microservice Vulnerabilities (20%)

## Overview

This domain focuses on utilizing Kubernetes mechanisms to isolate, protect, and control workloads. It covers Pod Security Standards, security contexts, secrets management, container runtime sandboxes, and mTLS implementation.

---

## 4.1 Set Up Appropriate OS-Level Security Domains

### Pod Security Standards (PSS)

Pod Security Standards define three levels of security policies:

| Level | Description |
|-------|-------------|
| **Privileged** | Unrestricted, allows known privilege escalations |
| **Baseline** | Minimally restrictive, prevents known privilege escalations |
| **Restricted** | Heavily restricted, follows security best practices |

### Pod Security Admission (PSA)

PSA is the built-in admission controller that enforces Pod Security Standards.

#### Admission Modes

| Mode | Behavior |
|------|----------|
| **enforce** | Policy violations cause pod rejection |
| **audit** | Violations logged but allowed |
| **warn** | Warnings shown to user but allowed |

#### Namespace Labels for PSA

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    # Enforce restricted level
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    
    # Audit at restricted level
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    
    # Warn at restricted level
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

#### Quick Namespace Setup Commands

```bash
# Add enforcement labels to existing namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# Create namespace with labels
kubectl create namespace secure-ns --dry-run=client -o yaml | \
  kubectl label --local -f - \
    pod-security.kubernetes.io/enforce=baseline \
    pod-security.kubernetes.io/warn=restricted \
    --dry-run=client -o yaml | \
  kubectl apply -f -
```

### Baseline Policy Requirements

The Baseline policy prevents known privilege escalations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-compliant-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      # Baseline requirements
      privileged: false                    # No privileged mode
      allowPrivilegeEscalation: false     # Recommended
    # No hostNetwork, hostPID, hostIPC
    # No hostPath volumes (except specific safe types)
    # Limited capabilities (no dangerous ones)
```

### Restricted Policy Requirements

The Restricted policy adds more constraints:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-compliant-pod
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
      capabilities:
        drop:
        - ALL
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

### Cluster-Wide PSA Configuration

Configure default PSA via AdmissionConfiguration:

```yaml
# /etc/kubernetes/admission/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
      - kube-system
```

Add to kube-apiserver:
```yaml
- --admission-control-config-file=/etc/kubernetes/admission/admission-config.yaml
```

---

## 4.2 Security Contexts

### Pod-Level Security Context

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
    supplementalGroups: [4000]
    seccompProfile:
      type: RuntimeDefault
    seLinuxOptions:
      level: "s0:c123,c456"
  containers:
  - name: app
    image: nginx
```

### Container-Level Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: container-security-context
spec:
  containers:
  - name: secure-container
    image: nginx
    securityContext:
      runAsUser: 1000
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
      seccompProfile:
        type: RuntimeDefault
```

### Linux Capabilities

```yaml
# Drop all capabilities and add only what's needed
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-demo
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE  # Allow binding to ports < 1024
        - CHOWN             # Allow changing file ownership
```

#### Common Linux Capabilities

| Capability | Description |
|------------|-------------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 |
| `NET_ADMIN` | Network configuration |
| `SYS_ADMIN` | Various admin operations (DANGEROUS) |
| `SYS_PTRACE` | Trace processes |
| `SYS_TIME` | Set system time |
| `CHOWN` | Change file ownership |
| `DAC_OVERRIDE` | Bypass file permission checks |
| `SETUID`/`SETGID` | Set UID/GID |

### Read-Only Root Filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-fs-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}
```

---

## 4.3 Manage Kubernetes Secrets

### Creating Secrets

```bash
# From literal values
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cr3tP@ss!'

# From files
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=/path/to/id_rsa \
  --from-file=ssh-publickey=/path/to/id_rsa.pub

# TLS secret
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com
```

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Generic secret (default) |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/service-account-token` | Service account token |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH authentication |

### Using Secrets in Pods

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    # Or load all keys as env vars
    envFrom:
    - secretRef:
        name: db-credentials
```

#### As Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400  # File permissions
      items:            # Optional: mount specific keys
      - key: username
        path: db-user
      - key: password
        path: db-pass
```

### Encryption at Rest

#### Create Encryption Configuration

```yaml
# /etc/kubernetes/enc/enc.yaml
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
      - identity: {}  # Fallback to unencrypted (for reading old secrets)
```

Generate a key:
```bash
head -c 32 /dev/urandom | base64
```

#### Configure API Server

```yaml
# Add to kube-apiserver manifest
- --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

Mount the configuration:
```yaml
volumeMounts:
- name: enc
  mountPath: /etc/kubernetes/enc
  readOnly: true
volumes:
- name: enc
  hostPath:
    path: /etc/kubernetes/enc
    type: DirectoryOrCreate
```

#### Verify Encryption

```bash
# Create a test secret
kubectl create secret generic test-secret --from-literal=key=value

# Read directly from etcd
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C

# Should see encrypted data, not plaintext
```

#### Re-encrypt Existing Secrets

```bash
# After enabling encryption, re-encrypt all secrets
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

### Secret Best Practices

1. **Enable encryption at rest**
2. **Use RBAC to restrict secret access**
3. **Don't log secrets** - Avoid printing secrets in logs
4. **Use short-lived secrets** - Rotate regularly
5. **Mount as files, not env vars** - Env vars are more easily leaked
6. **Use external secret management** - HashiCorp Vault, AWS Secrets Manager

---

## 4.4 Use Container Runtime Sandboxes

### RuntimeClass

RuntimeClass allows you to select the container runtime configuration for pods.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc  # Name of the handler on the node
scheduling:
  nodeSelector:
    sandbox: "gvisor"
---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-containers
handler: kata
```

### Using RuntimeClass in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-pod
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: nginx
```

### gVisor (runsc)

gVisor provides an application kernel written in Go that implements the Linux system call interface:

```yaml
# RuntimeClass for gVisor
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

### Kata Containers

Kata Containers runs workloads in lightweight VMs:

```yaml
# RuntimeClass for Kata Containers
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata-runtime
```

### When to Use Sandboxed Runtimes

- **Multi-tenant environments** - Isolate workloads from different tenants
- **Running untrusted code** - Extra isolation layer
- **Compliance requirements** - Stronger isolation mandates
- **Defense in depth** - Additional security layer

---

## 4.5 Implement Pod-to-Pod Encryption (mTLS)

### Using Cilium for mTLS

Cilium can provide transparent encryption between pods:

```yaml
# Enable encryption in Cilium config
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
  namespace: kube-system
data:
  enable-ipsec: "true"
  ipsec-key-file: /etc/ipsec/keys
```

### Using Istio Service Mesh for mTLS

#### Enable Strict mTLS

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

#### Namespace-Level mTLS

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Manual Certificate Management

For environments without service mesh:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mtls-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: certs
      mountPath: /etc/certs
      readOnly: true
  volumes:
  - name: certs
    secret:
      secretName: pod-tls-certs
```

---

## 4.6 Understand and Implement Isolation Techniques

### Multi-Tenancy Approaches

#### Namespace Isolation

```yaml
# Create isolated namespace
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    tenant: a
    pod-security.kubernetes.io/enforce: restricted
---
# Resource quota per tenant
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
# Limit range for pods
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

#### Network Isolation per Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Only pods in same namespace
```

### Node Isolation

```yaml
# Dedicated nodes for sensitive workloads
apiVersion: v1
kind: Pod
metadata:
  name: sensitive-pod
spec:
  nodeSelector:
    security-level: high
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "sensitive"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx
```

### PodAntiAffinity for Isolation

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: isolated-pod
  labels:
    app: sensitive-app
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: tenant
            operator: NotIn
            values:
            - trusted
        topologyKey: "kubernetes.io/hostname"
  containers:
  - name: app
    image: nginx
```

---

## Hands-On Lab Exercises

### Lab 1: Pod Security Standards Implementation

```bash
# Create namespace with PSA labels
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: psa-test
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
EOF

# Try to create a privileged pod (should fail)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: psa-test
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
EOF

# Create a compliant pod (should succeed)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: psa-test
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
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
EOF
```

### Lab 2: Secret Encryption at Rest

```bash
# Generate encryption key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# Create encryption config
mkdir -p /etc/kubernetes/enc
cat > /etc/kubernetes/enc/enc.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

# Update API server to use encryption config
# Add to /etc/kubernetes/manifests/kube-apiserver.yaml:
# - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml

# Wait for API server to restart
kubectl get pods -n kube-system -w

# Create test secret
kubectl create secret generic encryption-test --from-literal=mykey=myvalue

# Verify encryption in etcd
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/encryption-test
```

### Lab 3: Security Context Configuration

```bash
# Create pod with comprehensive security context
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: secure-container
    image: busybox
    command: ["sh", "-c", "id && ls -la /data && sleep infinity"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: data
      mountPath: /data
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: data
    emptyDir: {}
  - name: tmp
    emptyDir: {}
EOF

# Verify security settings
kubectl exec secure-pod -- id
kubectl exec secure-pod -- cat /proc/1/status | grep -i seccomp
```

---

## Practice Questions

1. **Configure a namespace** to enforce the `restricted` Pod Security Standard.

2. **Create a secret** from a file and mount it as a volume with file mode `0400`.

3. **Write a pod spec** that is compliant with the Restricted policy.

4. **How do you enable encryption at rest** for secrets in Kubernetes?

5. **Create a RuntimeClass** for gVisor and use it in a pod.

---

## Quick Reference

### Pod Security Standard Levels

| Control | Privileged | Baseline | Restricted |
|---------|------------|----------|------------|
| Host Namespaces | Allowed | Forbidden | Forbidden |
| Privileged | Allowed | Forbidden | Forbidden |
| Capabilities | All | Limited | Drop ALL |
| HostPath | Allowed | Allowed | Forbidden |
| runAsNonRoot | - | - | Required |
| Seccomp | - | - | RuntimeDefault/Localhost |

### Security Context Fields

| Field | Level | Description |
|-------|-------|-------------|
| `runAsUser` | Pod/Container | User ID |
| `runAsGroup` | Pod/Container | Primary group ID |
| `fsGroup` | Pod | Volume ownership group |
| `runAsNonRoot` | Pod/Container | Must run as non-root |
| `readOnlyRootFilesystem` | Container | Read-only root filesystem |
| `allowPrivilegeEscalation` | Container | Prevent privilege escalation |
| `privileged` | Container | Privileged mode |
| `capabilities` | Container | Linux capabilities |
| `seccompProfile` | Pod/Container | Seccomp profile |
| `seLinuxOptions` | Pod/Container | SELinux context |

### Secret Commands

```bash
# Create generic secret
kubectl create secret generic <name> --from-literal=key=value

# Create TLS secret
kubectl create secret tls <name> --cert=cert.pem --key=key.pem

# View secret (base64 encoded)
kubectl get secret <name> -o yaml

# Decode secret value
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d
```

---

## Official Documentation References

*These URLs are accessible during the CKS exam:*

- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)