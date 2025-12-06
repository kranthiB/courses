# CKS Domain 2: Cluster Hardening (15%)

## Overview

This domain focuses on protecting the Kubernetes API and utilizing RBAC effectively. It covers minimizing service account exposure, restricting API access, and keeping Kubernetes updated to avoid vulnerabilities.

---

## 2.1 Restrict Access to Kubernetes API

### Understanding API Server Access

The Kubernetes API server is the central management point for the cluster. All interactions with the cluster go through it:

```
User/Pod → Authentication → Authorization → Admission Control → API Server → etcd
```

### Securing API Server Configuration

Key flags in `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    # Authentication
    - --anonymous-auth=false
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    
    # Authorization
    - --authorization-mode=Node,RBAC
    
    # Admission Controllers
    - --enable-admission-plugins=NodeRestriction,PodSecurity
    
    # TLS
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    
    # Audit logging
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=100
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    
    # Encryption at rest
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
    
    # Disable insecure port (deprecated, but verify)
    - --insecure-port=0
```

### Authentication Methods

#### X.509 Client Certificates

```bash
# Generate a user certificate
openssl genrsa -out user1.key 2048
openssl req -new -key user1.key -out user1.csr -subj "/CN=user1/O=developers"

# Create CSR in Kubernetes
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user1-csr
spec:
  request: $(cat user1.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# Approve CSR
kubectl certificate approve user1-csr

# Get certificate
kubectl get csr user1-csr -o jsonpath='{.status.certificate}' | base64 -d > user1.crt

# Create kubeconfig for user
kubectl config set-credentials user1 \
  --client-certificate=user1.crt \
  --client-key=user1.key

kubectl config set-context user1-context \
  --cluster=kubernetes \
  --user=user1
```

#### Service Account Tokens

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
---
apiVersion: v1
kind: Secret
metadata:
  name: my-sa-token
  annotations:
    kubernetes.io/service-account.name: my-service-account
type: kubernetes.io/service-account-token
```

### Anonymous Authentication

Disable anonymous authentication:

```yaml
# In kube-apiserver manifest
- --anonymous-auth=false
```

To verify:

```bash
# Should return 401 Unauthorized
curl -sk https://<api-server>:6443/api/v1/namespaces
```

### API Server Network Access

Restrict API server access at the network level:

```yaml
# Example firewall rules (using cloud provider or iptables)
# Allow only specific IP ranges to access API server on port 6443
iptables -A INPUT -p tcp --dport 6443 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 6443 -j DROP
```

---

## 2.2 Use Role Based Access Controls to Minimize Exposure

### RBAC Components

| Resource | Scope | Purpose |
|----------|-------|---------|
| Role | Namespace | Define permissions within a namespace |
| ClusterRole | Cluster-wide | Define permissions cluster-wide |
| RoleBinding | Namespace | Bind Role/ClusterRole to subjects in namespace |
| ClusterRoleBinding | Cluster-wide | Bind ClusterRole to subjects cluster-wide |

### Creating a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]  # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### Creating a ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### Creating a RoleBinding

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
  name: my-service-account
  namespace: development
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Creating a ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: auditors
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Using ClusterRole with RoleBinding (Namespace-Scoped)

```yaml
# Use a ClusterRole but limit it to a specific namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: production  # Only applies to this namespace
subjects:
- kind: User
  name: dave
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole  # Using ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Verbs Reference

| Verb | Description |
|------|-------------|
| get | Read a specific resource |
| list | List resources |
| watch | Watch for changes |
| create | Create new resources |
| update | Update existing resources |
| patch | Partially update resources |
| delete | Delete resources |
| deletecollection | Delete multiple resources |
| impersonate | Impersonate users/groups |
| bind | Bind roles |
| escalate | Escalate privileges |

### Check RBAC Permissions

```bash
# Check what you can do
kubectl auth can-i create pods
kubectl auth can-i delete secrets --namespace kube-system

# Check what another user can do
kubectl auth can-i create pods --as jane
kubectl auth can-i list secrets --as system:serviceaccount:default:my-sa

# Check all permissions
kubectl auth can-i --list
kubectl auth can-i --list --as jane

# Check specific actions
kubectl auth can-i get pods --namespace production
kubectl auth can-i '*' '*'  # Check for admin access
```

### RBAC Best Practices

1. **Principle of Least Privilege**: Grant minimum required permissions
2. **Use Namespaced Roles**: Prefer Role over ClusterRole where possible
3. **Avoid Wildcards**: Don't use `*` for resources or verbs
4. **Regular Audits**: Periodically review RBAC configurations
5. **Don't Use cluster-admin**: Avoid ClusterRoleBinding to cluster-admin
6. **Avoid system:masters**: Never add users to system:masters group

### Dangerous RBAC Configurations to Avoid

```yaml
# DON'T DO THIS - Excessive permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bad-admin-binding
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin  # TOO POWERFUL
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# DON'T DO THIS - Wildcard permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dangerous-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

---

## 2.3 Exercise Caution in Using Service Accounts

### Default Service Account

Every namespace has a `default` service account. Pods use this automatically unless specified otherwise.

```bash
# View default service account
kubectl get sa default -o yaml

# View token (pre-1.24)
kubectl get secret $(kubectl get sa default -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 -d
```

### Disable Automatic Token Mounting

#### At ServiceAccount Level

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
automountServiceAccountToken: false
```

#### At Pod Level

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  automountServiceAccountToken: false
  containers:
  - name: main
    image: nginx
```

### Creating Minimal Service Accounts

```yaml
# Create service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
automountServiceAccountToken: false
---
# Create minimal role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: app-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]  # Specific resources only
  verbs: ["get"]
---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
```

### Token Request API (Kubernetes 1.22+)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-projected-token
spec:
  serviceAccountName: my-sa
  containers:
  - name: main
    image: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: token
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600  # Token expires in 1 hour
          audience: my-api
```

### Audit Service Account Usage

```bash
# List all service accounts
kubectl get sa -A

# Find pods using default service account
kubectl get pods -A -o jsonpath='{range .items[?(@.spec.serviceAccountName=="default")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'

# Check service account permissions
kubectl auth can-i --list --as=system:serviceaccount:default:default
```

---

## 2.4 Update Kubernetes Frequently

### Checking Current Versions

```bash
# Check cluster version
kubectl version

# Check node versions
kubectl get nodes -o wide

# Check component versions
kubectl get pods -n kube-system -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}'
```

### Upgrade Process with kubeadm

#### Control Plane Upgrade

```bash
# 1. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.31.0-00
apt-mark hold kubeadm

# 2. Verify upgrade plan
kubeadm upgrade plan

# 3. Apply upgrade
kubeadm upgrade apply v1.31.0

# 4. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.31.0-00 kubectl=1.31.0-00
apt-mark hold kubelet kubectl

# 5. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet
```

#### Worker Node Upgrade

```bash
# On control plane: Drain the node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# On worker node: Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.31.0-00
apt-mark hold kubeadm

# Upgrade node config
kubeadm upgrade node

# Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.31.0-00 kubectl=1.31.0-00
apt-mark hold kubelet kubectl

# Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# On control plane: Uncordon the node
kubectl uncordon <node-name>
```

### Version Skew Policy

- **kubelet**: Can be up to 2 minor versions behind API server
- **kube-controller-manager, kube-scheduler, cloud-controller-manager**: Must match API server
- **kubectl**: Can be 1 version ahead or behind API server

---

## 2.5 Restrict API Server Access to Specific IPs

### API Server CIDR Configuration

In cloud environments, use private endpoints:

```bash
# AWS EKS - Enable private endpoint
aws eks update-cluster-config \
  --name cluster-name \
  --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true

# GKE - Enable private cluster
gcloud container clusters create secure-cluster \
  --enable-private-endpoint \
  --enable-private-nodes \
  --master-ipv4-cidr 172.16.0.0/28
```

### API Server Authorized Networks

```bash
# GKE - Restrict master authorized networks
gcloud container clusters update secure-cluster \
  --enable-master-authorized-networks \
  --master-authorized-networks 10.0.0.0/8,192.168.1.0/24
```

---

## Hands-On Lab Exercises

### Lab 1: RBAC Configuration

```bash
# Create namespace
kubectl create namespace rbac-test

# Create service account
kubectl create sa developer-sa -n rbac-test

# Create Role for pod management
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: pod-manager
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF

# Create RoleBinding
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-manager
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: developer-sa
  namespace: rbac-test
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
EOF

# Test permissions
kubectl auth can-i list pods -n rbac-test --as=system:serviceaccount:rbac-test:developer-sa
kubectl auth can-i delete deployments -n rbac-test --as=system:serviceaccount:rbac-test:developer-sa
kubectl auth can-i list pods -n default --as=system:serviceaccount:rbac-test:developer-sa
```

### Lab 2: Secure Service Account Usage

```bash
# Create a pod without auto-mounted token
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-sa
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: secure-sa
  containers:
  - name: main
    image: nginx
EOF

# Verify no token is mounted
kubectl exec secure-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# Should show: ls: cannot access '/var/run/secrets/kubernetes.io/serviceaccount/': No such file or directory
```

### Lab 3: Create User with Certificate

```bash
# Generate key and CSR
openssl genrsa -out developer.key 2048
openssl req -new -key developer.key -out developer.csr -subj "/CN=developer/O=dev-team"

# Create CSR resource
CSR_CONTENT=$(cat developer.csr | base64 | tr -d '\n')
kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: developer-csr
spec:
  request: ${CSR_CONTENT}
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
  expirationSeconds: 86400
EOF

# Approve and get certificate
kubectl certificate approve developer-csr
kubectl get csr developer-csr -o jsonpath='{.status.certificate}' | base64 -d > developer.crt

# Setup kubeconfig
kubectl config set-credentials developer \
  --client-certificate=developer.crt \
  --client-key=developer.key

kubectl config set-context developer-context \
  --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --user=developer

# Test (will fail without RBAC)
kubectl --context=developer-context get pods
```

---

## Practice Questions

1. **Create a Role** named `deployment-manager` in namespace `app` that allows managing deployments (all verbs) but only reading pods.

2. **What command** checks if user `jane` can delete secrets in namespace `production`?

3. **Configure a ServiceAccount** named `minimal-sa` that does not automatically mount API credentials to pods.

4. **Create a ClusterRole** that allows reading nodes and persistent volumes, then bind it to a group called `ops-team`.

5. **How do you check** all permissions available to the current user?

---

## Quick Reference

### Common kubectl RBAC Commands

```bash
# Create role
kubectl create role <name> --verb=<verbs> --resource=<resources> -n <namespace>

# Create rolebinding
kubectl create rolebinding <name> --role=<role> --user=<user> -n <namespace>

# Create clusterrole
kubectl create clusterrole <name> --verb=<verbs> --resource=<resources>

# Create clusterrolebinding
kubectl create clusterrolebinding <name> --clusterrole=<role> --user=<user>

# Check permissions
kubectl auth can-i <verb> <resource> [--as=<user>] [-n <namespace>]
```

### Built-in ClusterRoles

| ClusterRole | Description |
|-------------|-------------|
| cluster-admin | Full access to all resources |
| admin | Full access within a namespace |
| edit | Read/write access to most resources in a namespace |
| view | Read-only access to most resources in a namespace |

### Service Account Token Paths

| Path | Content |
|------|---------|
| `/var/run/secrets/kubernetes.io/serviceaccount/token` | JWT token |
| `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt` | CA certificate |
| `/var/run/secrets/kubernetes.io/serviceaccount/namespace` | Namespace name |

---

## Official Documentation References

*These URLs are accessible during the CKS exam:*

- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)