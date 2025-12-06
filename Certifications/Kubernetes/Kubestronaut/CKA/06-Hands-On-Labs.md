# CKA Hands-On Labs

## Lab Environment Setup

Before starting, set up your practice environment:

### Option 1: Minikube (Single Node)
```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --nodes=3 --driver=docker

# Enable addons
minikube addons enable metrics-server
minikube addons enable ingress
```

### Option 2: Kind (Kubernetes in Docker)
```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create multi-node cluster
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

### Option 3: Killercoda (Free Browser-Based)
Visit https://killercoda.com/playgrounds/scenario/kubernetes

---

## Lab 1: Storage (10%)

### Lab 1.1: Create StorageClass and Dynamic Provisioning

**Objective:** Create a StorageClass and use it for dynamic PV provisioning.

**Tasks:**
1. Create a StorageClass named `fast-storage` with:
   - Provisioner: `kubernetes.io/no-provisioner`
   - Reclaim policy: Retain
   - Volume binding mode: WaitForFirstConsumer

2. Create a PersistentVolume named `pv-fast` with:
   - Capacity: 1Gi
   - Access mode: ReadWriteOnce
   - StorageClass: `fast-storage`
   - HostPath: `/mnt/data/fast`

3. Create a PersistentVolumeClaim named `pvc-fast` requesting 500Mi from the `fast-storage` class

4. Create a Pod named `storage-pod` that mounts this PVC at `/data`

**Execute these commands:**
```bash
# Create namespace for lab
kubectl create namespace lab-storage

# Set namespace
kubectl config set-context --current --namespace=lab-storage
```

<details>
<summary>Solution</summary>

```yaml
# 1. StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
---
# 2. PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-fast
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  hostPath:
    path: /mnt/data/fast
    type: DirectoryOrCreate
---
# 3. PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fast
  namespace: lab-storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  resources:
    requests:
      storage: 500Mi
---
# 4. Pod
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
  namespace: lab-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-fast
```

**Verification:**
```bash
kubectl get sc fast-storage
kubectl get pv pv-fast
kubectl get pvc pvc-fast
kubectl get pod storage-pod
kubectl exec storage-pod -- df -h /data
```
</details>

### Lab 1.2: Expand a PVC

**Objective:** Expand an existing PVC.

**Tasks:**
1. Create a StorageClass `expandable-sc` with `allowVolumeExpansion: true`
2. Create a PVC requesting 100Mi
3. Expand the PVC to 500Mi

<details>
<summary>Solution</summary>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-sc
provisioner: kubernetes.io/no-provisioner
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```bash
# Expand PVC
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"500Mi"}}}}'
```
</details>

---

## Lab 2: Troubleshooting (30%)

### Lab 2.1: Fix a Broken Pod

**Objective:** Debug and fix a pod that won't start.

**Setup - Create the broken pod:**
```bash
kubectl create namespace lab-troubleshoot

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
  namespace: lab-troubleshoot
spec:
  containers:
  - name: app
    image: nginx:nonexistent-tag
    ports:
    - containerPort: 80
EOF
```

**Tasks:**
1. Identify why the pod is not running
2. Fix the issue
3. Verify the pod is running

<details>
<summary>Solution</summary>

```bash
# 1. Check pod status
kubectl get pod broken-pod -n lab-troubleshoot
# Shows: ImagePullBackOff or ErrImagePull

# 2. Describe pod for details
kubectl describe pod broken-pod -n lab-troubleshoot
# Events show: Failed to pull image "nginx:nonexistent-tag"

# 3. Fix the image
kubectl set image pod/broken-pod app=nginx:latest -n lab-troubleshoot

# 4. Verify
kubectl get pod broken-pod -n lab-troubleshoot
# Should show: Running
```
</details>

### Lab 2.2: Fix a Broken Deployment

**Setup:**
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-deploy
  namespace: lab-troubleshoot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: different-label
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

**Tasks:**
1. Identify why pods are not being created
2. Fix the issue

<details>
<summary>Solution</summary>

```bash
# 1. Check deployment
kubectl get deploy broken-deploy -n lab-troubleshoot
kubectl describe deploy broken-deploy -n lab-troubleshoot
# Issue: selector doesn't match template labels

# 2. Fix - labels must match selector
kubectl patch deployment broken-deploy -n lab-troubleshoot --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/metadata/labels/app", "value": "web"}]'

# Or delete and recreate with correct labels
```
</details>

### Lab 2.3: Fix a Service with No Endpoints

**Setup:**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: lab-troubleshoot
  labels:
    app: web-app
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: lab-troubleshoot
spec:
  selector:
    app: wrong-label
  ports:
  - port: 80
    targetPort: 80
EOF
```

**Tasks:**
1. Find why the service has no endpoints
2. Fix the service

<details>
<summary>Solution</summary>

```bash
# 1. Check endpoints
kubectl get endpoints web-service -n lab-troubleshoot
# Shows: <none>

# 2. Check service selector
kubectl get svc web-service -n lab-troubleshoot -o jsonpath='{.spec.selector}'
# Shows: {"app":"wrong-label"}

# 3. Check pod labels
kubectl get pod web-pod -n lab-troubleshoot --show-labels
# Shows: app=web-app

# 4. Fix service selector
kubectl patch svc web-service -n lab-troubleshoot -p '{"spec":{"selector":{"app":"web-app"}}}'

# 5. Verify
kubectl get endpoints web-service -n lab-troubleshoot
# Should show pod IP
```
</details>

### Lab 2.4: Troubleshoot Node NotReady

**Objective:** Practice the troubleshooting steps for a NotReady node.

**Simulated scenario steps (study these):**
```bash
# 1. Check node status
kubectl get nodes
kubectl describe node <node-name>

# 2. SSH to node and check kubelet
ssh <node>
systemctl status kubelet
journalctl -u kubelet -n 100

# 3. Common fixes
systemctl restart kubelet
systemctl restart containerd

# 4. Check disk space
df -h

# 5. Check memory
free -m

# 6. Check kubelet config
cat /var/lib/kubelet/config.yaml
```

### Lab 2.5: Debug Application Logs

**Setup:**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
  namespace: lab-troubleshoot
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "App log: $(date)"; sleep 5; done']
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Sidecar log: $(date)"; sleep 3; done']
EOF
```

**Tasks:**
1. View logs from the `app` container
2. View logs from the `sidecar` container
3. Stream logs from both containers

<details>
<summary>Solution</summary>

```bash
# Logs from specific container
kubectl logs multi-container -c app -n lab-troubleshoot
kubectl logs multi-container -c sidecar -n lab-troubleshoot

# Stream logs
kubectl logs multi-container -c app -n lab-troubleshoot -f

# All containers
kubectl logs multi-container --all-containers=true -n lab-troubleshoot
```
</details>

---

## Lab 3: Workloads and Scheduling (15%)

### Lab 3.1: Rolling Update and Rollback

**Setup:**
```bash
kubectl create namespace lab-workloads
kubectl create deployment web --image=nginx:1.19 --replicas=4 -n lab-workloads
```

**Tasks:**
1. Update the deployment to use `nginx:1.20`
2. Watch the rollout status
3. View rollout history
4. Rollback to the previous version
5. Rollback to revision 1

<details>
<summary>Solution</summary>

```bash
# 1. Update image
kubectl set image deployment/web nginx=nginx:1.20 -n lab-workloads

# 2. Watch rollout
kubectl rollout status deployment/web -n lab-workloads

# 3. View history
kubectl rollout history deployment/web -n lab-workloads

# 4. Rollback to previous
kubectl rollout undo deployment/web -n lab-workloads

# 5. Rollback to revision 1
kubectl rollout undo deployment/web --to-revision=1 -n lab-workloads
```
</details>

### Lab 3.2: ConfigMaps and Secrets

**Tasks:**
1. Create a ConfigMap named `app-config` with:
   - `APP_ENV=production`
   - `LOG_LEVEL=info`

2. Create a Secret named `app-secret` with:
   - `DB_USER=admin`
   - `DB_PASS=supersecret`

3. Create a Pod that uses both as environment variables

<details>
<summary>Solution</summary>

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  -n lab-workloads

# Create Secret
kubectl create secret generic app-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=supersecret \
  -n lab-workloads
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
  namespace: lab-workloads
spec:
  containers:
  - name: app
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secret
```

```bash
# Verify
kubectl exec env-pod -n lab-workloads -- env | grep -E "APP_|DB_|LOG_"
```
</details>

### Lab 3.3: Node Affinity and Taints

**Tasks:**
1. Label a node with `disktype=ssd`
2. Create a pod that only runs on nodes with `disktype=ssd`
3. Taint a node with `dedicated=special:NoSchedule`
4. Create a pod that tolerates this taint

<details>
<summary>Solution</summary>

```bash
# 1. Label node
kubectl label nodes <node-name> disktype=ssd

# 3. Taint node
kubectl taint nodes <node-name> dedicated=special:NoSchedule
```

```yaml
# 2. Pod with node affinity
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  namespace: lab-workloads
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
---
# 4. Pod with toleration
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-pod
  namespace: lab-workloads
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "special"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
```
</details>

### Lab 3.4: Horizontal Pod Autoscaler

**Tasks:**
1. Create a deployment with resource requests
2. Create an HPA that scales between 2-10 replicas at 50% CPU

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deploy
  namespace: lab-workloads
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hpa-test
  template:
    metadata:
      labels:
        app: hpa-test
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

```bash
kubectl autoscale deployment hpa-deploy --cpu-percent=50 --min=2 --max=10 -n lab-workloads

# Verify
kubectl get hpa -n lab-workloads
```
</details>

---

## Lab 4: Cluster Architecture (25%)

### Lab 4.1: RBAC - Create User Access

**Tasks:**
1. Create a ServiceAccount named `dev-sa` in namespace `dev`
2. Create a Role that allows get, list, watch on pods
3. Bind the role to the ServiceAccount
4. Verify permissions

<details>
<summary>Solution</summary>

```bash
# Create namespace and service account
kubectl create namespace dev
kubectl create serviceaccount dev-sa -n dev

# Create role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

# Create rolebinding
kubectl create rolebinding dev-sa-pod-reader \
  --role=pod-reader \
  --serviceaccount=dev:dev-sa \
  -n dev

# Verify
kubectl auth can-i get pods --as=system:serviceaccount:dev:dev-sa -n dev
# Should return: yes

kubectl auth can-i delete pods --as=system:serviceaccount:dev:dev-sa -n dev
# Should return: no
```
</details>

### Lab 4.2: RBAC - ClusterRole

**Tasks:**
1. Create a ClusterRole that can read secrets across all namespaces
2. Bind it to a user named `auditor`

<details>
<summary>Solution</summary>

```bash
# Create ClusterRole
kubectl create clusterrole secret-reader \
  --verb=get,list,watch \
  --resource=secrets

# Create ClusterRoleBinding
kubectl create clusterrolebinding auditor-secrets \
  --clusterrole=secret-reader \
  --user=auditor

# Verify
kubectl auth can-i list secrets --as=auditor -A
```
</details>

### Lab 4.3: Backup and Restore etcd

**Tasks:**
1. Create an etcd backup to `/tmp/etcd-backup.db`
2. Verify the backup

<details>
<summary>Solution</summary>

```bash
# Find etcd pod and get certificate paths
kubectl get pods -n kube-system | grep etcd
kubectl describe pod etcd-<controlplane> -n kube-system | grep -A 5 Command

# Backup
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table
```
</details>

### Lab 4.4: Cluster Upgrade (Study Steps)

**Practice these steps conceptually:**
```bash
# 1. Check current version
kubectl get nodes
kubeadm version

# 2. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.31.0-*
apt-mark hold kubeadm

# 3. Plan upgrade
kubeadm upgrade plan

# 4. Apply upgrade (control plane)
kubeadm upgrade apply v1.31.0

# 5. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.0-* kubectl=1.31.0-*
apt-mark hold kubelet kubectl

# 6. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 7. For worker nodes
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# SSH to node, upgrade kubeadm, kubelet, kubectl
kubeadm upgrade node
kubectl uncordon <node>
```

---

## Lab 5: Services and Networking (20%)

### Lab 5.1: Create Services

**Setup:**
```bash
kubectl create namespace lab-network
kubectl create deployment nginx --image=nginx --replicas=3 -n lab-network
```

**Tasks:**
1. Expose as ClusterIP service on port 80
2. Create a NodePort service on port 30080
3. Test connectivity

<details>
<summary>Solution</summary>

```bash
# ClusterIP
kubectl expose deployment nginx --port=80 --target-port=80 --name=nginx-clusterip -n lab-network

# NodePort
kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort --name=nginx-nodeport -n lab-network

# Or with specific NodePort
kubectl create service nodeport nginx-np --tcp=80:80 --node-port=30080 -n lab-network

# Test ClusterIP
kubectl run test --image=busybox:1.28 --rm -it -n lab-network -- wget -qO- nginx-clusterip

# Test NodePort
curl <node-ip>:30080
```
</details>

### Lab 5.2: Network Policies

**Tasks:**
1. Create a default deny all ingress policy in namespace `lab-network`
2. Create a policy that allows ingress to pods with label `app=nginx` only from pods with label `access=true`

<details>
<summary>Solution</summary>

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: lab-network
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow specific ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-access
  namespace: lab-network
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
    ports:
    - protocol: TCP
      port: 80
```

```bash
# Test - should fail (no access label)
kubectl run test-no-access --image=busybox:1.28 --rm -it -n lab-network -- wget -qO- --timeout=2 nginx-clusterip

# Test - should succeed (has access label)
kubectl run test-access --image=busybox:1.28 --rm -it -n lab-network --labels="access=true" -- wget -qO- nginx-clusterip
```
</details>

### Lab 5.3: Ingress

**Tasks:**
1. Create an Ingress that routes `app.example.com` to a service

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: lab-network
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-clusterip
            port:
              number: 80
```

```bash
# Test (add to /etc/hosts or use curl with Host header)
curl -H "Host: app.example.com" http://<ingress-ip>
```
</details>

### Lab 5.4: DNS Troubleshooting

**Tasks:**
1. Verify CoreDNS is running
2. Test DNS resolution from a pod
3. Check DNS configuration

<details>
<summary>Solution</summary>

```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS
kubectl run dnstest --image=busybox:1.28 --rm -it -- nslookup kubernetes.default
kubectl run dnstest --image=busybox:1.28 --rm -it -- nslookup nginx-clusterip.lab-network.svc.cluster.local

# Check resolv.conf
kubectl run dnstest --image=busybox:1.28 --rm -it -- cat /etc/resolv.conf
```
</details>

---

## Lab Cleanup

```bash
kubectl delete namespace lab-storage lab-troubleshoot lab-workloads dev lab-network
kubectl delete clusterrole secret-reader
kubectl delete clusterrolebinding auditor-secrets
```

---

## Practice Schedule

| Day | Focus | Labs |
|-----|-------|------|
| 1-2 | Storage | Lab 1.1, 1.2 |
| 3-5 | Troubleshooting | Lab 2.1-2.5 |
| 6-7 | Workloads | Lab 3.1-3.4 |
| 8-10 | Cluster Architecture | Lab 4.1-4.4 |
| 11-12 | Networking | Lab 5.1-5.4 |
| 13-14 | Mixed Practice | All labs timed |