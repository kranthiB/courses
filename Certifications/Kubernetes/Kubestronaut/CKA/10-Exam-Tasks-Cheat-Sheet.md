# CKA Exam Tasks Cheat Sheet

## Quick Copy-Paste Solutions for Common Exam Tasks

**Print this or have it ready during practice.**

---

## 1. POD OPERATIONS

### Create Pod
```bash
k run nginx --image=nginx
k run nginx --image=nginx --port=80
k run nginx --image=nginx --labels="app=web,tier=frontend"
k run nginx --image=nginx -n mynamespace
k run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

### Pod with Command
```bash
k run busybox --image=busybox --command -- sleep 3600
k run busybox --image=busybox --command -- sh -c "echo hello; sleep 3600"
```

### Pod with Resources
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Pod with Probes
```yaml
spec:
  containers:
  - name: nginx
    image: nginx
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

### Multi-Container Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo hello; sleep 10; done']
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
```

### Init Container
```yaml
spec:
  initContainers:
  - name: init
    image: busybox
    command: ['sh', '-c', 'echo init done']
  containers:
  - name: app
    image: nginx
```

---

## 2. DEPLOYMENT OPERATIONS

### Create Deployment
```bash
k create deployment nginx --image=nginx --replicas=3
k create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml
```

### Scale
```bash
k scale deployment nginx --replicas=5
```

### Update Image
```bash
k set image deployment/nginx nginx=nginx:1.21
```

### Rollout
```bash
k rollout status deployment/nginx
k rollout history deployment/nginx
k rollout undo deployment/nginx
k rollout undo deployment/nginx --to-revision=2
k rollout pause deployment/nginx
k rollout resume deployment/nginx
```

### Deployment Strategy
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

---

## 3. SERVICE OPERATIONS

### Create Services
```bash
# ClusterIP
k expose deployment nginx --port=80 --target-port=80
k expose pod nginx --port=80

# NodePort
k expose deployment nginx --port=80 --type=NodePort
k expose deployment nginx --port=80 --type=NodePort --node-port=30080

# LoadBalancer
k expose deployment nginx --port=80 --type=LoadBalancer
```

### Service YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP  # or NodePort, LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    # nodePort: 30080  # for NodePort
```

### Check Endpoints
```bash
k get endpoints nginx-svc
k describe svc nginx-svc
```

---

## 4. CONFIGMAP & SECRET

### ConfigMap
```bash
k create configmap myconfig --from-literal=KEY1=value1 --from-literal=KEY2=value2
k create configmap myconfig --from-file=config.txt
k create configmap myconfig --from-file=configs/
```

### Secret
```bash
k create secret generic mysecret --from-literal=user=admin --from-literal=pass=secret
k create secret tls tls-secret --cert=tls.crt --key=tls.key
k create secret docker-registry regcred --docker-server=<server> --docker-username=<user> --docker-password=<pass>
```

### Use as Environment Variables
```yaml
spec:
  containers:
  - name: app
    image: nginx
    envFrom:
    - configMapRef:
        name: myconfig
    - secretRef:
        name: mysecret
    # Or specific keys:
    env:
    - name: MY_VAR
      valueFrom:
        configMapKeyRef:
          name: myconfig
          key: KEY1
    - name: MY_SECRET
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: pass
```

### Use as Volume
```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-vol
    configMap:
      name: myconfig
  - name: secret-vol
    secret:
      secretName: mysecret
```

### Decode Secret
```bash
k get secret mysecret -o jsonpath='{.data.pass}' | base64 -d
```

---

## 5. STORAGE

### PersistentVolume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-vol
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
```

### Use PVC in Pod
```yaml
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
      claimName: pvc-vol
```

### StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## 6. RBAC

### ServiceAccount
```bash
k create serviceaccount mysa
```

### Role
```bash
k create role pod-reader --verb=get,list,watch --resource=pods
k create role pod-reader --verb=get,list,watch --resource=pods --resource=services
```

### RoleBinding
```bash
k create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=default:mysa
k create rolebinding pod-reader-binding --role=pod-reader --user=jane
```

### ClusterRole
```bash
k create clusterrole node-reader --verb=get,list,watch --resource=nodes
```

### ClusterRoleBinding
```bash
k create clusterrolebinding node-reader-binding --clusterrole=node-reader --user=jane
```

### Check Permissions
```bash
k auth can-i create pods --as=jane
k auth can-i get pods --as=system:serviceaccount:default:mysa
k auth can-i --list --as=jane -n default
```

### Role YAML
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### RoleBinding YAML
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: mysa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 7. NETWORK POLICY

### Default Deny All Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### Default Deny All Egress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Allow Specific Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app
spec:
  podSelector:
    matchLabels:
      app: db
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

### Allow from Namespace
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: prod
    podSelector:
      matchLabels:
        app: backend
```

### Allow DNS Egress
```yaml
egress:
- ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
```

---

## 8. INGRESS

### Create Ingress
```bash
k create ingress myingress --rule="host/path=service:port"
k create ingress myingress --rule="app.com/*=svc:80" --rule="api.com/*=api-svc:8080"
```

### Ingress YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myingress
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
            name: web-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
```

### TLS Ingress
```yaml
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret
  rules:
  - host: app.example.com
    # ...
```

---

## 9. SCHEDULING

### Node Selector
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### Node Affinity
```yaml
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
```

### Taints and Tolerations
```bash
# Add taint
k taint nodes node1 key=value:NoSchedule

# Remove taint
k taint nodes node1 key=value:NoSchedule-
```

```yaml
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

### Label Node
```bash
k label node node1 disktype=ssd
k label node node1 disktype-  # remove
```

---

## 10. CLUSTER OPERATIONS

### Node Management
```bash
k cordon node1        # prevent scheduling
k uncordon node1      # allow scheduling
k drain node1 --ignore-daemonsets --delete-emptydir-data
```

### Context
```bash
k config get-contexts
k config use-context cluster1
k config set-context --current --namespace=myns
```

### etcd Backup
```bash
ETCDCTL_API=3 etcdctl snapshot save /tmp/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /tmp/backup.db --write-out=table
```

### etcd Restore
```bash
ETCDCTL_API=3 etcdctl snapshot restore /tmp/backup.db \
  --data-dir=/var/lib/etcd-restored
```

### Certificate Check
```bash
kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep "Not After"
```

---

## 11. DEBUGGING

### Logs
```bash
k logs pod-name
k logs pod-name -c container-name
k logs pod-name --previous
k logs pod-name -f
k logs -l app=nginx --all-containers
```

### Exec
```bash
k exec pod-name -- command
k exec -it pod-name -- /bin/sh
k exec -it pod-name -c container-name -- /bin/bash
```

### Port Forward
```bash
k port-forward pod/nginx 8080:80
k port-forward svc/nginx 8080:80
```

### Test Connectivity
```bash
k run test --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- svc:80
k run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup svc
k run test --image=busybox:1.28 --rm -it --restart=Never -- sh
```

### Describe
```bash
k describe pod pod-name
k describe node node-name
k describe svc svc-name
```

### Events
```bash
k get events --sort-by='.lastTimestamp'
k get events -A --sort-by='.lastTimestamp'
```

---

## 12. USEFUL JSONPATH

### Get Pod IPs
```bash
k get pods -o jsonpath='{.items[*].status.podIP}'
```

### Get Node IPs
```bash
k get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'
```

### Get Image Names
```bash
k get pods -o jsonpath='{.items[*].spec.containers[*].image}'
```

### Get Specific Field
```bash
k get pod nginx -o jsonpath='{.status.podIP}'
k get svc nginx -o jsonpath='{.spec.clusterIP}'
```

### Custom Columns
```bash
k get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName
```

---

## 13. JOB & CRONJOB

### Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 4
  template:
    spec:
      containers:
      - name: job
        image: busybox
        command: ["echo", "hello"]
      restartPolicy: Never
```

### CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mycron
spec:
  schedule: "*/5 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: job
            image: busybox
            command: ["echo", "hello"]
          restartPolicy: OnFailure
```

---

## 14. SECURITY CONTEXT

### Pod Security Context
```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

---

## 15. RESOURCE QUOTA & LIMIT RANGE

### Resource Quota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
```

### Limit Range
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
spec:
  limits:
  - default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

---

## QUICK ALIASES

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias kds='kubectl describe svc'
alias kdd='kubectl describe deploy'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kn='kubectl config set-context --current --namespace'

export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
```

---

## EXAM DAY CHECKLIST

1. ✅ Read question carefully - note namespace and context
2. ✅ Switch context: `k config use-context <context>`
3. ✅ Set namespace if needed: `kn <namespace>`
4. ✅ Use imperative commands when possible
5. ✅ Generate YAML with `--dry-run=client -o yaml`
6. ✅ Verify after creation
7. ✅ Don't get stuck - flag and move on
8. ✅ Use kubernetes.io/docs for YAML templates