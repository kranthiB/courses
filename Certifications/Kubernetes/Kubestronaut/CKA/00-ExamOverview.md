# CKA Exam Overview and Quick Reference Guide

## Exam Information

| Attribute | Details |
|-----------|---------|
| **Duration** | 2 hours |
| **Format** | Performance-based (hands-on, command-line) |
| **Questions** | 15-20 tasks |
| **Passing Score** | 66% |
| **Kubernetes Version** | v1.34 (as of 2025) |
| **Retakes** | One free retake included |
| **Validity** | 2 years |
| **Cost** | $445 USD |

---

## Domain Weights

| Domain | Weight | Study Guide File |
|--------|--------|------------------|
| Troubleshooting | 30% | `05-Domain5-Troubleshooting.md` |
| Cluster Architecture, Installation & Configuration | 25% | `01-Domain1-ClusterArchitectureInstallationandConfiguration.md` |
| Services & Networking | 20% | `03-Domain3-ServicesandNetworking.md` |
| Workloads & Scheduling | 15% | `02-Domain2-WorkloadsandScheduling.md` |
| Storage | 10% | `04-Domain4-Storage.md` |

---

## Exam Environment

### Allowed Resources

During the exam, you can access:
- https://kubernetes.io/docs/
- https://kubernetes.io/blog/
- https://helm.sh/docs/

**Tip:** Bookmark important pages before the exam!

### Terminal Environment

- Multiple clusters provided
- Context switching required: `kubectl config use-context <context-name>`
- `kubectl` is pre-installed with bash completion
- `alias k=kubectl` is available
- Browser-based terminal (one tab for exam, one for docs)

---

## Essential kubectl Commands

### Setup Aliases (Already available but useful to remember)

```bash
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
export do='--dry-run=client -o yaml'
```

### Context and Namespace

```bash
# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Set default namespace
kubectl config set-context --current --namespace=<namespace>
```

### Quick Resource Creation

```bash
# Pod
kubectl run nginx --image=nginx

# Deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Service (ClusterIP)
kubectl expose deployment nginx --port=80 --target-port=80

# Service (NodePort)
kubectl expose deployment nginx --port=80 --type=NodePort

# ConfigMap
kubectl create configmap myconfig --from-literal=key=value

# Secret
kubectl create secret generic mysecret --from-literal=password=secret123

# ServiceAccount
kubectl create serviceaccount mysa

# Role
kubectl create role pod-reader --verb=get,list --resource=pods

# RoleBinding
kubectl create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=default:mysa
```

### Generate YAML Templates

```bash
# Generate YAML without creating
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml > service.yaml
```

### Get Resources

```bash
# Basic get
kubectl get pods,svc,deploy -A

# Wide output with more details
kubectl get pods -o wide

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# JSON path
kubectl get pod nginx -o jsonpath='{.status.podIP}'

# Sort by
kubectl get pods --sort-by=.metadata.creationTimestamp
```

### Debugging

```bash
# Describe (events at bottom)
kubectl describe pod <pod-name>

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>
kubectl logs <pod-name> --previous

# Exec into pod
kubectl exec -it <pod-name> -- /bin/sh

# Port forward
kubectl port-forward pod/<pod-name> 8080:80

# Test connectivity
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- <service>:<port>
kubectl run test --image=busybox:1.28 --rm -it -- nslookup <service>
```

---

## Quick Reference by Domain

### Storage (10%)

```bash
# StorageClass
kubectl get sc
kubectl describe sc <name>

# PersistentVolume
kubectl get pv
kubectl describe pv <name>

# PersistentVolumeClaim
kubectl get pvc
kubectl describe pvc <name>
```

Key concepts:
- Access modes: RWO, ROX, RWX, RWOP
- Reclaim policies: Retain, Delete
- Dynamic provisioning via StorageClass

### Troubleshooting (30%)

```bash
# Cluster health
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info

# Node troubleshooting
kubectl describe node <node>
ssh <node> "systemctl status kubelet"
ssh <node> "journalctl -u kubelet"

# Pod troubleshooting
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous

# Events
kubectl get events --sort-by='.lastTimestamp'

# Control plane (kubeadm)
ls /etc/kubernetes/manifests/
```

### Workloads & Scheduling (15%)

```bash
# Deployments
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Scaling
kubectl scale deployment <name> --replicas=5

# Autoscaling
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=50

# Scheduling
kubectl label nodes <node> key=value
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key=value:NoSchedule-  # Remove
```

### Cluster Architecture (25%)

```bash
# RBAC
kubectl auth can-i <verb> <resource> --as=<user>
kubectl create role <name> --verb=get,list --resource=pods
kubectl create rolebinding <name> --role=<role> --user=<user>

# kubeadm
kubeadm init
kubeadm join
kubeadm upgrade plan
kubeadm upgrade apply v1.xx.x
kubeadm token create --print-join-command

# etcd backup
ETCDCTL_API=3 etcdctl snapshot save <path> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Services & Networking (20%)

```bash
# Services
kubectl get svc
kubectl get endpoints
kubectl describe svc <name>

# Network Policies
kubectl get netpol
kubectl describe netpol <name>

# DNS
kubectl run test --image=busybox:1.28 --rm -it -- nslookup <service>

# Ingress
kubectl get ingress
kubectl describe ingress <name>
```

---

## Time Management Strategy

| Priority | Focus Area | Time Allocation |
|----------|------------|-----------------|
| High | Easy questions (quick wins) | First 30 minutes |
| Medium | Medium difficulty | Next 60 minutes |
| Low | Hard/time-consuming | Last 30 minutes |

### Tips:

1. **Read the question carefully** - Note the namespace and context
2. **Check context first** - `kubectl config use-context <context>`
3. **Use imperative commands** - Faster than writing YAML from scratch
4. **Use `--dry-run=client -o yaml`** - Generate templates
5. **Bookmark key docs** - Have URLs ready
6. **Flag and skip** - Don't get stuck on one question
7. **Validate your work** - Verify resources are created correctly

---

## Common Mistakes to Avoid

1. **Wrong context/namespace** - Always verify before starting a task
2. **Typos in YAML** - Use `kubectl apply --dry-run=server` to validate
3. **Missing labels** - Services need matching selectors
4. **Wrong ports** - Distinguish between `port`, `targetPort`, `nodePort`
5. **Forgetting to restart kubelet** - After config changes
6. **Not reading events** - `kubectl describe` events section is crucial
7. **Time management** - Don't spend too long on one question

---

## Must-Know YAML Templates

### Pod with Resources and Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
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
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: myconfig
```

### Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Role and RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Final Checklist

Before the exam:
- [ ] Practice with killer.sh simulator (included with exam)
- [ ] Know kubectl commands by heart
- [ ] Practice kubeadm cluster setup
- [ ] Practice etcd backup and restore
- [ ] Understand Network Policies
- [ ] Know RBAC configuration
- [ ] Practice troubleshooting scenarios
- [ ] Bookmark important documentation pages
- [ ] Test your exam environment (webcam, browser)

During the exam:
- [ ] Switch to correct context for each question
- [ ] Set namespace if specified
- [ ] Verify resources after creation
- [ ] Use imperative commands when possible
- [ ] Don't get stuck - flag and move on
- [ ] Use remaining time to review flagged questions

---

## Study Resources

### Official Resources
- Kubernetes Documentation: https://kubernetes.io/docs/
- CNCF Curriculum: https://github.com/cncf/curriculum
- Killer.sh Simulator (included with exam purchase)

### Practice
- Set up a local cluster with kubeadm or minikube
- Practice scenarios from each domain
- Time yourself on practice questions

---

**Good luck on your CKA exam!**