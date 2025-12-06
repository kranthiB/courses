# CKA Study Guide: Cluster Architecture, Installation and Configuration (25%)

## Domain Overview

This domain covers the second-largest portion of the CKA exam (25%). It tests your knowledge of Kubernetes cluster architecture, RBAC, cluster setup with kubeadm, lifecycle management, high availability, Helm, Kustomize, extension interfaces (CNI, CSI, CRI), Custom Resource Definitions (CRDs), and Operators.

---

## 1. Manage Role-Based Access Control (RBAC)

### 1.1 RBAC Overview

RBAC regulates access to resources based on the roles assigned to users, groups, or service accounts.

#### RBAC API Objects

| Object | Scope | Description |
|--------|-------|-------------|
| `Role` | Namespace | Defines permissions within a namespace |
| `ClusterRole` | Cluster-wide | Defines permissions across the cluster |
| `RoleBinding` | Namespace | Binds Role to users/groups/serviceaccounts |
| `ClusterRoleBinding` | Cluster-wide | Binds ClusterRole to users/groups/serviceaccounts |

### 1.2 Creating Roles

#### Namespace-Scoped Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]          # "" = core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

#### ClusterRole

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

#### Available Verbs

| Verb | Description |
|------|-------------|
| `get` | Read a specific resource |
| `list` | List resources |
| `watch` | Watch for changes |
| `create` | Create new resources |
| `update` | Update existing resources |
| `patch` | Partially update resources |
| `delete` | Delete resources |
| `deletecollection` | Delete collection of resources |

### 1.3 Creating Bindings

#### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-service-account
  namespace: default
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: managers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 1.4 Imperative RBAC Commands

```bash
# Create Role
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n default

# Create ClusterRole
kubectl create clusterrole secret-reader --verb=get,list --resource=secrets

# Create RoleBinding
kubectl create rolebinding read-pods --role=pod-reader --user=jane -n default
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:mysa -n default

# Create ClusterRoleBinding
kubectl create clusterrolebinding read-secrets --clusterrole=secret-reader --user=admin

# Check permissions
kubectl auth can-i create pods --as=jane
kubectl auth can-i get secrets --as=system:serviceaccount:default:mysa
kubectl auth can-i --list --as=jane -n default
```

### 1.5 ServiceAccounts

```bash
# Create ServiceAccount
kubectl create serviceaccount my-service-account -n default

# Get ServiceAccount token (K8s 1.24+)
kubectl create token my-service-account

# Assign ServiceAccount to Pod
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  automountServiceAccountToken: true  # or false to disable
  containers:
  - name: app
    image: nginx
```

### 1.6 RBAC Best Practices

1. **Principle of Least Privilege**: Grant minimum necessary permissions
2. **Use namespaces**: Isolate resources and RBAC
3. **Avoid `cluster-admin`**: Only use when necessary
4. **Regular audits**: Review RoleBindings periodically
5. **Use Groups**: Manage users through groups rather than individually
6. **Avoid wildcards**: Don't use `*` for resources or verbs unless necessary

---

## 2. Prepare Infrastructure for Installing a Kubernetes Cluster

### 2.1 Node Requirements

#### Control Plane Node

- 2+ CPUs
- 2GB+ RAM
- Container runtime (containerd, CRI-O)
- Network connectivity between nodes
- Unique hostname, MAC address, product_uuid

#### Worker Node

- 1+ CPUs
- 1GB+ RAM
- Container runtime
- Network connectivity to control plane

### 2.2 Prerequisites

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Verify
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### 2.3 Install Container Runtime (containerd)

```bash
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 2.4 Install kubeadm, kubelet, kubectl

```bash
# Add Kubernetes repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install packages
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet
```

---

## 3. Create and Manage Kubernetes Clusters Using kubeadm

### 3.1 Initialize Control Plane

```bash
# Basic initialization
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# With custom API server address
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<control-plane-ip> \
  --control-plane-endpoint=<load-balancer-ip>:6443

# Generate config file
kubeadm config print init-defaults > kubeadm-config.yaml

# Initialize with config file
sudo kubeadm init --config=kubeadm-config.yaml
```

### 3.2 Post-Initialization Setup

```bash
# Configure kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Or for root
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 3.3 Install Network Plugin

```bash
# Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Weave
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

### 3.4 Join Worker Nodes

```bash
# On control plane, get join command
kubeadm token create --print-join-command

# On worker node
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### 3.5 Token Management

```bash
# List tokens
kubeadm token list

# Create new token
kubeadm token create

# Create token with join command
kubeadm token create --print-join-command

# Delete token
kubeadm token delete <token-value>
```

---

## 4. Manage the Lifecycle of Kubernetes Clusters

### 4.1 Cluster Upgrades

#### Upgrade Control Plane

```bash
# Check available versions
apt-cache madison kubeadm

# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.31.0-*
sudo apt-mark hold kubeadm

# Plan upgrade
sudo kubeadm upgrade plan

# Apply upgrade
sudo kubeadm upgrade apply v1.31.0

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.0-* kubectl=1.31.0-*
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Upgrade Worker Nodes

```bash
# On control plane: drain the node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# On worker node: upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.31.0-*
sudo apt-mark hold kubeadm

# Upgrade node config
sudo kubeadm upgrade node

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.0-* kubectl=1.31.0-*
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

# On control plane: uncordon the node
kubectl uncordon <node-name>
```

### 4.2 Node Management

```bash
# Cordon node (prevent new pods)
kubectl cordon <node-name>

# Drain node (evict pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon node
kubectl uncordon <node-name>

# Remove node from cluster
kubectl delete node <node-name>

# On the node being removed
sudo kubeadm reset
```

### 4.3 Backup and Restore etcd

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table

# Restore etcd (stop kubelet first)
sudo systemctl stop kubelet

ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380 \
  --name=master

# Update etcd manifest to use new data directory
# Edit /etc/kubernetes/manifests/etcd.yaml

sudo systemctl start kubelet
```

### 4.4 Certificate Management

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Renew all certificates
sudo kubeadm certs renew all

# Renew specific certificate
sudo kubeadm certs renew apiserver

# View certificate details
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

---

## 5. Implement and Configure a Highly-Available Control Plane

### 5.1 HA Architecture Components

- **Multiple control plane nodes** (3+ recommended)
- **Load balancer** for API server (HAProxy, nginx, cloud LB)
- **Stacked etcd** (on control plane nodes) or **External etcd**

### 5.2 HA Cluster Setup

```bash
# Initialize first control plane with load balancer endpoint
sudo kubeadm init \
  --control-plane-endpoint "LOAD_BALANCER_DNS:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Join additional control plane nodes
sudo kubeadm join LOAD_BALANCER_DNS:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>

# Join worker nodes
sudo kubeadm join LOAD_BALANCER_DNS:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### 5.3 Load Balancer Configuration (HAProxy Example)

```
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master1 192.168.1.10:6443 check
    server master2 192.168.1.11:6443 check
    server master3 192.168.1.12:6443 check
```

---

## 6. Use Helm and Kustomize to Install Cluster Components

### 6.1 Helm Basics

Helm is a package manager for Kubernetes.

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo nginx
helm search hub nginx

# Install chart
helm install my-nginx bitnami/nginx

# Install with custom values
helm install my-nginx bitnami/nginx -f values.yaml
helm install my-nginx bitnami/nginx --set replicaCount=3

# List releases
helm list

# Show values
helm show values bitnami/nginx

# Upgrade release
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Rollback
helm rollback my-nginx 1

# Uninstall
helm uninstall my-nginx

# Generate manifests without installing
helm template my-nginx bitnami/nginx > manifests.yaml
```

### 6.2 Kustomize Basics

Kustomize is built into kubectl for customizing Kubernetes manifests.

#### Directory Structure

```
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

#### Base kustomization.yaml

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

#### Overlay for Production

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: prod-
namespace: production
replicas:
  - name: my-app
    count: 5
images:
  - name: nginx
    newTag: 1.22.0
patches:
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 512Mi
    target:
      kind: Deployment
      name: my-app
```

#### Kustomize Commands

```bash
# Preview output
kubectl kustomize overlays/prod/

# Apply directly
kubectl apply -k overlays/prod/

# Build and save to file
kubectl kustomize overlays/prod/ > output.yaml
```

---

## 7. Understand Extension Interfaces (CNI, CSI, CRI)

### 7.1 Container Network Interface (CNI)

CNI provides networking for containers.

| Plugin | Features |
|--------|----------|
| Flannel | Simple overlay network |
| Calico | Network policies, BGP routing |
| Weave | Mesh networking |
| Cilium | eBPF-based, advanced security |

```bash
# CNI configuration location
/etc/cni/net.d/

# CNI plugin binaries
/opt/cni/bin/

# Check CNI configuration
cat /etc/cni/net.d/*.conf
```

### 7.2 Container Storage Interface (CSI)

CSI provides storage integration for containers.

```yaml
# Example CSI StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-storage
provisioner: csi.example.com
parameters:
  type: ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 7.3 Container Runtime Interface (CRI)

CRI defines the interface between kubelet and container runtimes.

| Runtime | Socket |
|---------|--------|
| containerd | `/run/containerd/containerd.sock` |
| CRI-O | `/var/run/crio/crio.sock` |

```bash
# Configure kubelet to use specific runtime
# /var/lib/kubelet/config.yaml
containerRuntimeEndpoint: unix:///run/containerd/containerd.sock

# Verify container runtime
crictl info
crictl ps
```

---

## 8. Understand CRDs, Install and Configure Operators

### 8.1 Custom Resource Definitions (CRDs)

CRDs extend the Kubernetes API with custom resources.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

```bash
# Create CRD
kubectl apply -f crd.yaml

# List CRDs
kubectl get crd

# Create custom resource
kubectl apply -f - <<EOF
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: my-cron
spec:
  cronSpec: "* * * * */5"
  image: my-app:latest
  replicas: 3
EOF

# Get custom resources
kubectl get crontabs
kubectl get ct
```

### 8.2 Operators

Operators extend Kubernetes functionality by combining CRDs with controllers.

```bash
# Install Operator Lifecycle Manager (OLM)
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.25.0/install.sh | bash -s v0.25.0

# Install an operator (example: Prometheus)
kubectl create -f https://operatorhub.io/install/prometheus.yaml

# List installed operators
kubectl get csv -n operators
```

---

## 9. Practice Exercises

### Exercise 1: Create RBAC for a User

Create a Role that allows reading pods and deployments in the `dev` namespace, and bind it to user `developer`.

<details>
<summary>Solution</summary>

```bash
kubectl create role dev-reader --verb=get,list,watch --resource=pods,deployments -n dev
kubectl create rolebinding dev-reader-binding --role=dev-reader --user=developer -n dev

# Verify
kubectl auth can-i get pods --as=developer -n dev
```

</details>

### Exercise 2: Upgrade Cluster with kubeadm

Outline the steps to upgrade a cluster from 1.30.0 to 1.31.0.

<details>
<summary>Solution</summary>

```bash
# 1. Upgrade control plane
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.31.0-*
sudo apt-mark hold kubeadm
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.0

# 2. Upgrade kubelet and kubectl on control plane
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.0-* kubectl=1.31.0-*
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 3. For each worker node:
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# SSH to node, upgrade kubeadm, run 'kubeadm upgrade node', upgrade kubelet/kubectl
kubectl uncordon <node>
```

</details>

### Exercise 3: Backup etcd

Backup etcd to `/opt/etcd-backup.db`.

<details>
<summary>Solution</summary>

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table
```

</details>

---

## 10. Key Points for the Exam

1. **RBAC**: Know Role vs ClusterRole, RoleBinding vs ClusterRoleBinding
2. **kubeadm**: init, join, upgrade, reset commands
3. **etcd backup/restore**: ETCDCTL_API=3 and certificate paths
4. **Cluster upgrades**: Correct order (kubeadm → kubelet → kubectl)
5. **Node management**: drain, cordon, uncordon
6. **Helm**: install, upgrade, rollback, template
7. **Kustomize**: Built into kubectl, patches, overlays
8. **CRDs**: How to create and use custom resources
9. **Extension interfaces**: CNI for networking, CSI for storage, CRI for runtime

---

## 11. Documentation References

- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
- https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- https://helm.sh/docs/
- https://kubectl.docs.kubernetes.io/references/kustomize/