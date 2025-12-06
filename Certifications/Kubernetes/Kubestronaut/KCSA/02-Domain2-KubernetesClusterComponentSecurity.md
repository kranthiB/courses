# KCSA Exam Study Guide: Domain 2 - Kubernetes Cluster Component Security

## Domain Weight: 22% (~13 questions out of 60)

This domain focuses on securing all Kubernetes components: control plane, worker nodes, networking, and storage.

---

## 2.1 Control Plane Component Security

### Control Plane Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                                │
│                                                                     │
│  ┌───────────────┐  ┌──────────────┐  ┌─────────────────────────┐   │
│  │ kube-apiserver│  │    etcd      │  │ kube-controller-manager │   │
│  │               │  │              │  │                         │   │
│  │ • AuthN/AuthZ │  │ • Encrypted  │  │ • Service Account Keys  │   │
│  │ • Admission   │  │ • mTLS       │  │ • Root CA               │   │
│  │ • Audit Logs  │  │ • Access     │  │ • Secure Defaults       │   │
│  │ • TLS         │  │   Control    │  │                         │   │
│  └───────────────┘  └──────────────┘  └─────────────────────────┘   │
│                                                                     │
│  ┌──────────────┐  ┌───────────────────────────────────────────┐    │
│  │kube-scheduler│  │        cloud-controller-manager           │    │
│  │              │  │                                           │    │
│  │ • Secure     │  │ • Cloud Provider Credentials              │    │
│  │   Config     │  │ • IAM Integration                         │    │
│  └──────────────┘  └───────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### API Server Security

The API Server is the **most critical component** - it's the front door to your cluster.

**Authentication Methods**:
| Method | Description | Use Case |
|--------|-------------|----------|
| X.509 Client Certs | Certificate-based auth | Service accounts, admins |
| Bearer Tokens | Token-based auth | Service accounts |
| OIDC | OpenID Connect | Enterprise SSO |
| Webhook | External auth service | Custom authentication |
| Bootstrap Tokens | Initial cluster setup | Node joining |

**Key Security Configurations**:
```yaml
# API Server security flags
--anonymous-auth=false                    # Disable anonymous access
--authorization-mode=RBAC,Node            # Enable RBAC and Node authorization
--enable-admission-plugins=...            # Enable admission controllers
--audit-log-path=/var/log/audit.log       # Enable audit logging
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--tls-cert-file=/path/to/cert.pem         # TLS certificate
--tls-private-key-file=/path/to/key.pem   # TLS private key
--client-ca-file=/path/to/ca.pem          # Client CA for mTLS
--encryption-provider-config=/path/to/encryption-config.yaml  # Encrypt secrets at rest
```

**API Server Best Practices**:
- Never expose API server to the public internet
- Use TLS for all communication
- Enable audit logging
- Restrict anonymous authentication
- Use RBAC authorization
- Enable admission controllers
- Encrypt secrets at rest

### etcd Security

etcd stores ALL cluster data including secrets - it's the most sensitive component.

**Security Requirements**:
```
┌─────────────────────────────────────────────────────────┐
│                    etcd Security                        │
├─────────────────────────────────────────────────────────┤
│ ✓ Encrypt data at rest                                  │
│ ✓ Use mTLS for peer communication                       │
│ ✓ Use mTLS for client communication                     │
│ ✓ Restrict network access                               │
│ ✓ Run on dedicated nodes                                │
│ ✓ Regular backups (encrypted)                           │
│ ✓ Separate CA from Kubernetes CA                        │
│ ✓ Limit who can access etcd directly                    │
└─────────────────────────────────────────────────────────┘
```

**etcd Security Flags**:
```yaml
--cert-file=/path/to/server.crt
--key-file=/path/to/server.key
--trusted-ca-file=/path/to/ca.crt
--client-cert-auth=true
--peer-cert-file=/path/to/peer.crt
--peer-key-file=/path/to/peer.key
--peer-trusted-ca-file=/path/to/peer-ca.crt
--peer-client-cert-auth=true
```

**Critical**: Anyone with direct access to etcd can read ALL secrets in plaintext!

### Controller Manager Security

**Security Configurations**:
```yaml
--use-service-account-credentials=true    # Use per-controller credentials
--service-account-private-key-file=/path/to/sa.key
--root-ca-file=/path/to/ca.crt
--bind-address=127.0.0.1                  # Bind to localhost only
--profiling=false                          # Disable profiling
```

### Scheduler Security

**Security Configurations**:
```yaml
--bind-address=127.0.0.1                  # Bind to localhost only
--profiling=false                          # Disable profiling
--authentication-kubeconfig=/path/to/kubeconfig
--authorization-kubeconfig=/path/to/kubeconfig
```

---

## 2.2 Worker Node Component Security

### Worker Node Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         WORKER NODE                                 │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐    │
│  │   kubelet    │  │  kube-proxy  │  │   Container Runtime     │    │
│  │              │  │              │  │                         │    │
│  │ • AuthN/AuthZ│  │ • iptables/  │  │ • containerd/CRI-O      │    │
│  │ • TLS        │  │   IPVS       │  │ • Socket Security       │    │
│  │ • Read-only  │  │ • Config     │  │ • Rootless Mode         │    │
│  │   Port       │  │   Security   │  │                         │    │
│  └──────────────┘  └──────────────┘  └─────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                         Pods                                │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │    │
│  │  │Container│  │Container│  │Container│  │Container│         │    │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘         │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Kubelet Security

The kubelet runs on every node and manages containers - a compromised kubelet = compromised node.

**Security Configurations**:
```yaml
# kubelet security flags
--anonymous-auth=false                     # Disable anonymous access
--authorization-mode=Webhook               # Use API server for authz
--client-ca-file=/path/to/ca.crt          # Verify client certs
--tls-cert-file=/path/to/kubelet.crt      # TLS certificate
--tls-private-key-file=/path/to/kubelet.key
--rotate-certificates=true                 # Auto-rotate certs
--protect-kernel-defaults=true             # Protect kernel settings
--read-only-port=0                         # Disable read-only port
--streaming-connection-idle-timeout=5m     # Timeout idle connections
--make-iptables-util-chains=true
```

**Kubelet API Protection**:
- Port 10250: Main kubelet API (requires authentication)
- Port 10255: Read-only port (DISABLE THIS - `--read-only-port=0`)

### Container Runtime Security

**containerd Security**:
```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri"]
  enable_selinux = true
  
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

**Runtime Security Features**:
| Feature | Purpose |
|---------|---------|
| Rootless containers | Run containers without root |
| User namespaces | Map container root to unprivileged user |
| Seccomp | Restrict system calls |
| AppArmor/SELinux | Mandatory access control |
| Read-only rootfs | Prevent filesystem modification |

### kube-proxy Security

**Security Configurations**:
```yaml
--bind-address=127.0.0.1                  # Bind metrics to localhost
--metrics-bind-address=127.0.0.1:10249
--healthz-bind-address=127.0.0.1:10256
```

---

## 2.3 Pod Security

### Pod Specification Security

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  
  # Service account
  serviceAccountName: minimal-sa
  automountServiceAccountToken: false    # Don't mount SA token if not needed
  
  containers:
  - name: app
    image: myapp:1.0@sha256:abc123...    # Use digest, not tag
    
    # Container-level security context
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
          - ALL
    
    # Resource limits (prevent DoS)
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    
    # Volume mounts
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  
  volumes:
  - name: tmp
    emptyDir: {}
```

### Security Context Options

| Option | Level | Purpose |
|--------|-------|---------|
| `runAsUser` | Pod/Container | Run as specific UID |
| `runAsGroup` | Pod/Container | Run as specific GID |
| `runAsNonRoot` | Pod/Container | Prevent running as root |
| `fsGroup` | Pod | Group for volume ownership |
| `allowPrivilegeEscalation` | Container | Prevent privilege escalation |
| `readOnlyRootFilesystem` | Container | Read-only root filesystem |
| `capabilities` | Container | Linux capabilities |
| `seccompProfile` | Pod/Container | Seccomp filtering |
| `seLinuxOptions` | Pod/Container | SELinux labels |
| `privileged` | Container | Full host access (AVOID!) |

### Dangerous Pod Configurations (AVOID!)

```yaml
# ❌ DANGEROUS - DO NOT USE IN PRODUCTION
spec:
  hostNetwork: true        # Access host network
  hostPID: true            # Access host processes
  hostIPC: true            # Access host IPC
  containers:
  - name: dangerous
    securityContext:
      privileged: true     # Full host access
      capabilities:
        add: ["SYS_ADMIN"] # Dangerous capability
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /              # Mount entire host filesystem
```

---

## 2.4 Container Networking Security

### Network Model

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Networking                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pod-to-Pod:     All pods can communicate (by default)          │
│  Pod-to-Service: Via kube-proxy (iptables/IPVS)                 │
│  External:       Via Services (NodePort, LoadBalancer, Ingress) │
│                                                                 │
│  CNI Plugin handles: IP allocation, routing, network policies   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### CNI Security Features

| CNI Plugin | Network Policy | Encryption | Additional Security |
|------------|---------------|------------|---------------------|
| **Calico** | Yes | WireGuard | Host endpoint protection |
| **Cilium** | Yes | WireGuard/IPsec | eBPF-based, L7 policies |
| **Weave** | Yes | IPsec | Encrypted overlay |
| **Flannel** | No (basic) | No | Simple overlay |

### Network Policy Deep Dive

**Default Deny All**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # Select all pods
  policyTypes:
  - Ingress
  - Egress
```

**Allow Specific Traffic**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
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
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 8080
```

**Allow Egress to DNS**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

---

## 2.5 Client Security (kubeconfig)

### kubeconfig Structure

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <base64-ca-cert>
    server: https://kubernetes.example.com:6443
  name: production
contexts:
- context:
    cluster: production
    user: admin
    namespace: default
  name: production-admin
current-context: production-admin
users:
- name: admin
  user:
    client-certificate-data: <base64-client-cert>
    client-key-data: <base64-client-key>
```

### kubeconfig Security Best Practices

| Practice | Description |
|----------|-------------|
| Protect the file | Set permissions to 600 (`chmod 600 ~/.kube/config`) |
| Don't share | Each user should have their own kubeconfig |
| Use short-lived certs | Rotate certificates regularly |
| Avoid embedding secrets | Use external credential providers |
| Use contexts | Separate contexts for different clusters/namespaces |
| Audit access | Log kubectl commands |

### Credential Providers

```yaml
# Using exec-based credential provider
users:
- name: gke-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: gke-gcloud-auth-plugin
      installHint: Install gke-gcloud-auth-plugin
```

---

## 2.6 Storage Security

### Storage Security Considerations

```
┌─────────────────────────────────────────────────────────────────┐
│                      Storage Security                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Data at Rest:                                                  │
│  • Encrypt PersistentVolumes                                    │
│  • Use encrypted storage classes                                │
│  • Encrypt etcd (contains ConfigMaps, Secrets)                  │
│                                                                 │
│  Access Control:                                                │
│  • RBAC for PV/PVC creation                                     │
│  • Namespace isolation for PVCs                                 │
│  • fsGroup for volume permissions                               │
│                                                                 │
│  Volume Types Security:                                         │
│  • Avoid hostPath in production                                 │
│  • Use CSI drivers with encryption                              │
│  • Implement volume snapshots for backup                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Dangerous Volume Types

```yaml
# ❌ AVOID in production - hostPath can access host filesystem
volumes:
- name: host-volume
  hostPath:
    path: /etc
    type: Directory

# ✅ SAFER - Use PersistentVolumeClaim
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

### Encrypted Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: arn:aws:kms:region:account:key/key-id
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

---

## Key Exam Tips for This Domain

1. **API Server is the most critical** - know all security configurations
2. **etcd contains ALL secrets** - encryption and access control are essential
3. **Kubelet port 10255** is read-only and should be disabled
4. **Know dangerous pod configurations**: hostNetwork, hostPID, privileged
5. **Network Policies default to allow all** - explicit deny required
6. **kubeconfig security**: permissions, rotation, don't share
7. **Avoid hostPath volumes** in production

---

## Practice Questions

1. Which port should be disabled on the kubelet?
   - Answer: Port 10255 (read-only port)

2. What stores all Kubernetes cluster state including secrets?
   - Answer: etcd

3. Which flag enables RBAC authorization on the API server?
   - Answer: `--authorization-mode=RBAC`

4. What securityContext option prevents a container from gaining more privileges?
   - Answer: `allowPrivilegeEscalation: false`

5. What is the default network policy behavior in Kubernetes?
   - Answer: Allow all traffic (no restrictions)

6. Which volume type should be avoided in production due to security risks?
   - Answer: hostPath

7. What file permissions should be set on kubeconfig?
   - Answer: 600 (read/write for owner only)

8. Which component runs on every node and manages containers?
   - Answer: kubelet

---

## Additional Resources

- [Kubernetes Component Security](https://kubernetes.io/docs/concepts/security/)
- [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Pod Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)