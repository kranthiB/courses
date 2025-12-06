# CKA Mock Exam Scenarios

## Exam Simulation Instructions

- **Total Time:** 2 hours
- **Questions:** 17 tasks (similar to real exam)
- **Passing Score:** 66%
- **Environment:** Use your practice cluster
- **Resources:** Only kubernetes.io/docs allowed

### Before Starting:
1. Set a timer for 2 hours
2. Have kubernetes.io/docs open in another tab
3. Don't look at solutions until time is up
4. Note your time for each question

---

## Mock Exam 1

### Context Setup
```bash
# Create namespaces for this mock exam
kubectl create namespace alpha
kubectl create namespace beta
kubectl create namespace gamma
kubectl create namespace delta
```

---

### Question 1 (4%) - Pod Creation
**Context:** `kubectl config use-context cluster1`
**Namespace:** `alpha`

Create a pod named `web-pod` with the following specifications:
- Image: `nginx:1.21`
- Labels: `app=web`, `tier=frontend`
- Container port: 80
- Resource requests: CPU 100m, Memory 128Mi
- Resource limits: CPU 200m, Memory 256Mi

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: alpha
  labels:
    app: web
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```

```bash
kubectl apply -f web-pod.yaml
```
</details>

---

### Question 2 (4%) - Deployment and Service
**Context:** `kubectl config use-context cluster1`
**Namespace:** `beta`

Create a deployment named `api-server` with:
- Image: `nginx:1.20`
- Replicas: 3
- Labels: `app=api`

Then expose it as a ClusterIP service named `api-service` on port 80.

<details>
<summary>Solution</summary>

```bash
kubectl create deployment api-server --image=nginx:1.20 --replicas=3 -n beta
kubectl label deployment api-server app=api -n beta --overwrite
kubectl expose deployment api-server --name=api-service --port=80 -n beta
```
</details>

---

### Question 3 (7%) - RBAC
**Context:** `kubectl config use-context cluster1`
**Namespace:** `gamma`

1. Create a ServiceAccount named `app-sa`
2. Create a Role named `pod-manager` that allows:
   - get, list, watch, create, delete on pods
   - get, list on services
3. Bind the role to the ServiceAccount

<details>
<summary>Solution</summary>

```bash
kubectl create serviceaccount app-sa -n gamma

kubectl create role pod-manager \
  --verb=get,list,watch,create,delete --resource=pods \
  --verb=get,list --resource=services \
  -n gamma

kubectl create rolebinding app-sa-pod-manager \
  --role=pod-manager \
  --serviceaccount=gamma:app-sa \
  -n gamma

# Verify
kubectl auth can-i create pods --as=system:serviceaccount:gamma:app-sa -n gamma
```
</details>

---

### Question 4 (7%) - Network Policy
**Context:** `kubectl config use-context cluster1`
**Namespace:** `delta`

Create a NetworkPolicy named `db-policy` that:
- Applies to pods with label `app=database`
- Allows ingress only from pods with label `app=backend`
- Allows ingress only on port 5432
- Denies all other ingress traffic

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: delta
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```
</details>

---

### Question 5 (5%) - ConfigMap and Secret
**Context:** `kubectl config use-context cluster1`
**Namespace:** `alpha`

1. Create a ConfigMap named `app-config` with:
   - `DATABASE_HOST=db.example.com`
   - `DATABASE_PORT=5432`

2. Create a Secret named `app-secret` with:
   - `DB_USER=admin`
   - `DB_PASSWORD=secretpass123`

3. Create a pod named `config-pod` using image `nginx` that mounts:
   - The ConfigMap as environment variables
   - The Secret as a volume at `/etc/secrets`

<details>
<summary>Solution</summary>

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=db.example.com \
  --from-literal=DATABASE_PORT=5432 \
  -n alpha

kubectl create secret generic app-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=secretpass123 \
  -n alpha
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
  namespace: alpha
spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```
</details>

---

### Question 6 (7%) - Storage
**Context:** `kubectl config use-context cluster1`

1. Create a PersistentVolume named `pv-data` with:
   - Capacity: 2Gi
   - Access mode: ReadWriteOnce
   - HostPath: `/mnt/data`
   - StorageClass: manual

2. Create a PersistentVolumeClaim named `pvc-data` in namespace `beta` requesting 1Gi from StorageClass `manual`

3. Create a pod named `storage-pod` in namespace `beta` that mounts the PVC at `/data`

<details>
<summary>Solution</summary>

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /mnt/data
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
  namespace: beta
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
---
# Pod
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
  namespace: beta
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-data
```
</details>

---

### Question 7 (7%) - Troubleshooting - Fix Deployment
**Context:** `kubectl config use-context cluster1`
**Namespace:** `gamma`

A deployment has been created but pods are not running. Fix the issue.

```bash
# Create the broken deployment
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: gamma
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken
  template:
    metadata:
      labels:
        app: working
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

<details>
<summary>Solution</summary>

```bash
# Identify the issue - selector doesn't match template labels
kubectl describe deployment broken-app -n gamma

# Fix by patching the template labels to match selector
kubectl patch deployment broken-app -n gamma --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/metadata/labels/app", "value": "broken"}]'

# Verify
kubectl get pods -n gamma -l app=broken
```
</details>

---

### Question 8 (4%) - Rolling Update
**Context:** `kubectl config use-context cluster1`
**Namespace:** `beta`

Update the `api-server` deployment to use image `nginx:1.21` and record the change.

Then rollback to the previous version.

<details>
<summary>Solution</summary>

```bash
# Update with record
kubectl set image deployment/api-server nginx=nginx:1.21 -n beta
kubectl annotate deployment/api-server kubernetes.io/change-cause="Updated to nginx:1.21" -n beta

# Verify
kubectl rollout status deployment/api-server -n beta

# Rollback
kubectl rollout undo deployment/api-server -n beta
```
</details>

---

### Question 9 (7%) - Node Maintenance
**Context:** `kubectl config use-context cluster1`

Schedule node `worker-1` for maintenance:
1. Make the node unschedulable
2. Evict all pods from the node
3. After maintenance (simulated), make the node schedulable again

<details>
<summary>Solution</summary>

```bash
# Make unschedulable
kubectl cordon worker-1

# Evict pods
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# After maintenance - make schedulable
kubectl uncordon worker-1
```
</details>

---

### Question 10 (7%) - Ingress
**Context:** `kubectl config use-context cluster1`
**Namespace:** `alpha`

Create an Ingress named `web-ingress` that:
- Uses IngressClass `nginx`
- Routes `web.example.com/` to service `web-service` port 80
- Routes `web.example.com/api` to service `api-service` port 8080

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: alpha
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```
</details>

---

### Question 11 (7%) - Scheduling
**Context:** `kubectl config use-context cluster1`
**Namespace:** `delta`

1. Label node `worker-2` with `disk=ssd`
2. Create a pod named `ssd-pod` with image `nginx` that:
   - Only schedules on nodes with `disk=ssd`
   - Has toleration for taint `dedicated=special:NoSchedule`

<details>
<summary>Solution</summary>

```bash
kubectl label node worker-2 disk=ssd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  namespace: delta
spec:
  nodeSelector:
    disk: ssd
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

---

### Question 12 (5%) - Logs and Debugging
**Context:** `kubectl config use-context cluster1`
**Namespace:** `gamma`

1. Get the logs from pod `app-sa` container `nginx` and save to `/tmp/pod-logs.txt`
2. Find all pods in the cluster that are not in Running state and save to `/tmp/non-running-pods.txt`

<details>
<summary>Solution</summary>

```bash
# Get logs
kubectl logs <pod-name> -c nginx -n gamma > /tmp/pod-logs.txt

# Find non-running pods
kubectl get pods -A --field-selector=status.phase!=Running -o wide > /tmp/non-running-pods.txt
```
</details>

---

### Question 13 (7%) - etcd Backup
**Context:** `kubectl config use-context cluster1`

Create a backup of etcd and save it to `/tmp/etcd-backup.db`

The etcd is running with:
- Endpoint: https://127.0.0.1:2379
- CA cert: /etc/kubernetes/pki/etcd/ca.crt
- Server cert: /etc/kubernetes/pki/etcd/server.crt
- Server key: /etc/kubernetes/pki/etcd/server.key

<details>
<summary>Solution</summary>

```bash
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table
```
</details>

---

### Question 14 (4%) - ClusterRole
**Context:** `kubectl config use-context cluster1`

Create a ClusterRole named `node-reader` that can:
- get, list, watch nodes

Create a ClusterRoleBinding to bind this role to user `auditor`

<details>
<summary>Solution</summary>

```bash
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes

kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=auditor

# Verify
kubectl auth can-i list nodes --as=auditor
```
</details>

---

### Question 15 (5%) - Service DNS
**Context:** `kubectl config use-context cluster1`
**Namespace:** `beta`

Create a test pod and verify that you can resolve the `api-service` via DNS.
Save the full DNS name to `/tmp/service-dns.txt`

<details>
<summary>Solution</summary>

```bash
# Test DNS
kubectl run dnstest --image=busybox:1.28 --rm -it -n beta --restart=Never -- nslookup api-service

# Full DNS name
echo "api-service.beta.svc.cluster.local" > /tmp/service-dns.txt

# Or get it dynamically
kubectl run dnstest --image=busybox:1.28 --rm -it -n beta --restart=Never -- \
  nslookup api-service.beta.svc.cluster.local
```
</details>

---

### Question 16 (7%) - Multi-container Pod
**Context:** `kubectl config use-context cluster1`
**Namespace:** `alpha`

Create a pod named `multi-pod` with two containers:
1. Container `app`: image `nginx`, port 80
2. Container `sidecar`: image `busybox`, command `['sh', '-c', 'while true; do echo "Sidecar running"; sleep 10; done']`

Both containers should share a volume mounted at `/shared`

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
  namespace: alpha
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Sidecar running"; sleep 10; done']
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
```
</details>

---

### Question 17 (6%) - Troubleshooting Service
**Context:** `kubectl config use-context cluster1`
**Namespace:** `delta`

A service `web-svc` exists but has no endpoints. Debug and fix the issue.

```bash
# Setup
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  namespace: delta
  labels:
    app: webapp
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
  name: web-svc
  namespace: delta
spec:
  selector:
    app: wrong-app
  ports:
  - port: 80
    targetPort: 80
EOF
```

<details>
<summary>Solution</summary>

```bash
# Check endpoints
kubectl get endpoints web-svc -n delta
# Shows no endpoints

# Check service selector
kubectl get svc web-svc -n delta -o jsonpath='{.spec.selector}'
# Shows: {"app":"wrong-app"}

# Check pod labels
kubectl get pod web-app -n delta --show-labels
# Shows: app=webapp

# Fix the service selector
kubectl patch svc web-svc -n delta -p '{"spec":{"selector":{"app":"webapp"}}}'

# Verify
kubectl get endpoints web-svc -n delta
```
</details>

---

## Scoring

| Question | Points | Your Score |
|----------|--------|------------|
| Q1 | 4% | |
| Q2 | 4% | |
| Q3 | 7% | |
| Q4 | 7% | |
| Q5 | 5% | |
| Q6 | 7% | |
| Q7 | 7% | |
| Q8 | 4% | |
| Q9 | 7% | |
| Q10 | 7% | |
| Q11 | 7% | |
| Q12 | 5% | |
| Q13 | 7% | |
| Q14 | 4% | |
| Q15 | 5% | |
| Q16 | 7% | |
| Q17 | 6% | |
| **Total** | **100%** | |

**Passing Score: 66%**

---

## Mock Exam 2

### Question 1 (5%) - Create Job
**Namespace:** `default`

Create a Job named `pi-job` that:
- Uses image `perl:5.34`
- Runs command: `perl -Mbignum=bpi -wle 'print bpi(2000)'`
- Completes 5 times
- Runs 2 in parallel
- Has backoff limit of 4

<details>
<summary>Solution</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 4
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```
</details>

---

### Question 2 (7%) - CronJob
**Namespace:** `default`

Create a CronJob named `backup-job` that:
- Runs every day at 2:00 AM
- Uses image `busybox`
- Runs command: `echo "Backup completed at $(date)"`
- Keeps 3 successful job histories
- Keeps 1 failed job history

<details>
<summary>Solution</summary>

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command: ["/bin/sh", "-c", "echo 'Backup completed at $(date)'"]
          restartPolicy: OnFailure
```
</details>

---

### Question 3 (7%) - Liveness and Readiness Probes
**Namespace:** `default`

Create a pod named `health-pod` with:
- Image: `nginx`
- Liveness probe: HTTP GET on path `/healthz`, port 80, initial delay 15s, period 10s
- Readiness probe: HTTP GET on path `/ready`, port 80, initial delay 5s, period 5s

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-pod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```
</details>

---

### Question 4 (7%) - DaemonSet
**Namespace:** `kube-system`

Create a DaemonSet named `log-collector` that:
- Uses image `fluentd:latest`
- Runs on all nodes including control plane (tolerate control plane taint)
- Mounts host path `/var/log` to `/var/log` in container

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: log-collector
  template:
    metadata:
      labels:
        name: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```
</details>

---

### Question 5 (7%) - Static Pod
**Node:** control plane node

Create a static pod named `static-web` on the control plane node with:
- Image: `nginx`
- Port: 80

<details>
<summary>Solution</summary>

```bash
# SSH to control plane node
# Create manifest in static pod directory

cat <<EOF > /etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF

# Verify
kubectl get pods -A | grep static-web
```
</details>

---

### Question 6 (7%) - HPA
**Namespace:** `default`

1. Create a deployment named `cpu-stress` with:
   - Image: `nginx`
   - Replicas: 1
   - Resource requests: CPU 100m, Memory 128Mi

2. Create an HPA that:
   - Targets the deployment
   - Scales between 1 and 10 replicas
   - Targets 50% CPU utilization

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-stress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-stress
  template:
    metadata:
      labels:
        app: cpu-stress
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
```

```bash
kubectl autoscale deployment cpu-stress --cpu-percent=50 --min=1 --max=10
```
</details>

---

### Question 7 (5%) - Service Account Token
**Namespace:** `default`

1. Create a ServiceAccount named `app-sa`
2. Create a pod named `token-pod` that uses this ServiceAccount
3. Do NOT automatically mount the service account token

<details>
<summary>Solution</summary>

```bash
kubectl create serviceaccount app-sa
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: token-pod
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
  containers:
  - name: nginx
    image: nginx
```
</details>

---

### Question 8 (7%) - Pod Security Context
**Namespace:** `default`

Create a pod named `secure-pod` with:
- Image: `nginx`
- Run as user ID 1000
- Run as group ID 3000
- Filesystem group: 2000
- Read-only root filesystem
- Do not allow privilege escalation

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: nginx
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
```
</details>

---

### Question 9 (5%) - Resource Quota
**Namespace:** Create new namespace `limited`

Create a ResourceQuota named `compute-quota` with:
- Max 10 pods
- Max CPU requests: 4
- Max Memory requests: 4Gi
- Max CPU limits: 8
- Max Memory limits: 8Gi

<details>
<summary>Solution</summary>

```bash
kubectl create namespace limited
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: limited
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
```
</details>

---

### Question 10 (7%) - Network Policy - Egress
**Namespace:** `default`

Create a NetworkPolicy named `egress-policy` that:
- Applies to pods with label `app=frontend`
- Allows egress only to:
  - Pods with label `app=backend` on port 8080
  - DNS (UDP port 53 to any pod)
- Denies all other egress

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-policy
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
  - ports:
    - protocol: UDP
      port: 53
```
</details>

---

### Question 11 (7%) - Upgrade Cluster (Documentation)
**Context:** `cluster1`

Document the steps to upgrade a kubeadm cluster from v1.30 to v1.31.

List the commands for:
1. Upgrading the control plane
2. Upgrading worker nodes

<details>
<summary>Solution</summary>

```bash
# Control Plane Upgrade Steps:
# 1. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.31.0-*
apt-mark hold kubeadm

# 2. Verify upgrade plan
kubeadm upgrade plan

# 3. Apply upgrade
kubeadm upgrade apply v1.31.0

# 4. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.0-* kubectl=1.31.0-*
apt-mark hold kubelet kubectl

# 5. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# Worker Node Upgrade Steps:
# 1. Drain node (from control plane)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. On worker node - upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.31.0-*
apt-mark hold kubeadm

# 3. Upgrade node
kubeadm upgrade node

# 4. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.0-* kubectl=1.31.0-*
apt-mark hold kubelet kubectl

# 5. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 6. Uncordon node (from control plane)
kubectl uncordon <node-name>
```
</details>

---

### Question 12 (5%) - Certificate Check
**Context:** `cluster1`

Check when the apiserver certificate expires and save the expiration date to `/tmp/apiserver-cert-expiry.txt`

<details>
<summary>Solution</summary>

```bash
# Using kubeadm
kubeadm certs check-expiration | grep apiserver > /tmp/apiserver-cert-expiry.txt

# Or using openssl
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep "Not After" > /tmp/apiserver-cert-expiry.txt
```
</details>

---

### Question 13 (7%) - Init Container
**Namespace:** `default`

Create a pod named `init-pod` with:
- Init container: image `busybox`, command to create file `/work-dir/ready`
- Main container: image `nginx`, that only starts after init container completes
- Both containers share a volume at `/work-dir`

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
  - name: init
    image: busybox
    command: ['sh', '-c', 'touch /work-dir/ready']
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  volumes:
  - name: workdir
    emptyDir: {}
```
</details>

---

### Question 14 (5%) - JSONPath
**Context:** `cluster1`

Get all node names and their internal IPs using jsonpath. Save to `/tmp/node-ips.txt`

Format: `<node-name>: <internal-ip>`

<details>
<summary>Solution</summary>

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}' > /tmp/node-ips.txt
```
</details>

---

### Question 15 (6%) - Troubleshoot Kubelet

A worker node shows NotReady status. Document the troubleshooting steps.

<details>
<summary>Solution</summary>

```bash
# 1. Check node status
kubectl get nodes
kubectl describe node <node-name>

# 2. SSH to the node
ssh <node>

# 3. Check kubelet status
systemctl status kubelet

# 4. Check kubelet logs
journalctl -u kubelet -n 100
journalctl -u kubelet --since "10 minutes ago"

# 5. Check container runtime
systemctl status containerd
crictl ps

# 6. Common fixes:
# - Restart kubelet
systemctl restart kubelet

# - Restart container runtime
systemctl restart containerd

# - Check certificates
ls -la /var/lib/kubelet/pki/

# - Check kubelet config
cat /var/lib/kubelet/config.yaml

# - Check disk space
df -h

# - Check memory
free -m
```
</details>

---

## Cleanup

```bash
kubectl delete namespace alpha beta gamma delta limited
kubectl delete clusterrole node-reader
kubectl delete clusterrolebinding node-reader-binding
kubectl delete pv pv-data
```

---

## Practice Tips

1. **Time yourself**: Each mock exam should take < 2 hours
2. **Don't look at solutions**: Try to solve completely first
3. **Review mistakes**: Understand why you got something wrong
4. **Repeat**: Do each mock exam at least 3 times
5. **Mix it up**: Create your own variations of questions