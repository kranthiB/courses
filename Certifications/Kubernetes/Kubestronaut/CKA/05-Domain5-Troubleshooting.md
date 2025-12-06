# CKA Study Guide: Troubleshooting (30%)

## Domain Overview

Troubleshooting is the **largest domain** on the CKA exam, covering 30% of the total score. This domain tests your ability to diagnose and resolve issues in Kubernetes clusters, including cluster and node problems, component failures, application issues, and networking problems.

---

## 1. Troubleshoot Clusters and Nodes

### 1.1 Node Status and Conditions

#### Checking Node Status

```bash
# List all nodes with status
kubectl get nodes

# Get detailed node information
kubectl describe node <node-name>

# Get node conditions specifically
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Check node resource usage
kubectl top nodes
```

#### Node Conditions

| Condition | Description |
|-----------|-------------|
| `Ready` | Node is healthy and ready to accept Pods |
| `MemoryPressure` | Node is running low on memory |
| `DiskPressure` | Node is running low on disk space |
| `PIDPressure` | Too many processes on the node |
| `NetworkUnavailable` | Node's network is not configured correctly |

### 1.2 Common Node Issues and Solutions

#### Node Not Ready

```bash
# Check node status
kubectl describe node <node-name>

# Common causes:
# 1. Kubelet not running
ssh <node> "systemctl status kubelet"
ssh <node> "systemctl restart kubelet"

# 2. Container runtime issues
ssh <node> "systemctl status containerd"  # or docker
ssh <node> "crictl ps"

# 3. Network plugin issues
ssh <node> "systemctl status kubelet"
# Check CNI plugin logs

# 4. Certificate issues
ssh <node> "openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text -noout"
```

#### Node Resource Exhaustion

```bash
# Check disk usage
ssh <node> "df -h"

# Check memory usage
ssh <node> "free -m"

# Check for evicted pods
kubectl get pods --all-namespaces --field-selector=status.phase=Failed | grep Evicted

# Clear completed/failed pods
kubectl delete pods --field-selector=status.phase=Failed --all-namespaces
```

### 1.3 Kubelet Troubleshooting

```bash
# Check kubelet status
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -f
journalctl -u kubelet --since "10 minutes ago"

# Check kubelet configuration
cat /var/lib/kubelet/config.yaml

# Common kubelet issues:
# - Unable to connect to API server
# - Certificate expiration
# - Incorrect cluster DNS configuration
# - Container runtime socket issues
```

---

## 2. Troubleshoot Cluster Components

### 2.1 Control Plane Components

The control plane consists of:
- **kube-apiserver**: API entry point
- **etcd**: Cluster data store
- **kube-scheduler**: Pod scheduling
- **kube-controller-manager**: Controllers
- **cloud-controller-manager**: Cloud provider integration (if applicable)

#### Checking Control Plane Health

```bash
# If components run as static Pods (kubeadm clusters)
kubectl get pods -n kube-system

# Check component status
kubectl get componentstatuses  # Deprecated but may still work

# Check API server health
kubectl cluster-info
curl -k https://localhost:6443/healthz

# Check etcd health
kubectl -n kube-system exec -it etcd-<node> -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

### 2.2 Static Pod Locations

For kubeadm clusters, control plane manifests are at:

```bash
/etc/kubernetes/manifests/
├── etcd.yaml
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

### 2.3 Troubleshooting API Server

```bash
# Check API server pod
kubectl -n kube-system get pod kube-apiserver-<controlplane-node>
kubectl -n kube-system logs kube-apiserver-<controlplane-node>

# Check API server manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Common issues:
# - Certificate expiration
# - etcd connectivity
# - Incorrect flags in manifest
# - Port binding issues

# Check certificates
kubeadm certs check-expiration
```

### 2.4 Troubleshooting etcd

```bash
# Check etcd pod
kubectl -n kube-system logs etcd-<controlplane-node>

# Check etcd member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check etcd database size
ETCDCTL_API=3 etcdctl endpoint status --write-out=table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 2.5 Troubleshooting Scheduler

```bash
# Check scheduler logs
kubectl -n kube-system logs kube-scheduler-<controlplane-node>

# Common scheduler issues:
# - Pod stuck in Pending (scheduler not running)
# - Misconfigured scheduler policies
# - Resource constraints

# Check why pod is not scheduled
kubectl describe pod <pod-name> | grep -A 10 Events
```

### 2.6 Troubleshooting Controller Manager

```bash
# Check controller manager logs
kubectl -n kube-system logs kube-controller-manager-<controlplane-node>

# Common issues:
# - Service accounts not being created
# - ReplicaSets not scaling
# - Node lifecycle issues
```

---

## 3. Monitor Cluster and Application Resource Usage

### 3.1 Metrics Server

```bash
# Check if metrics server is installed
kubectl get deployment metrics-server -n kube-system

# Install metrics server (if needed)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# View node resource usage
kubectl top nodes

# View pod resource usage
kubectl top pods --all-namespaces
kubectl top pods -n <namespace>

# Sort by CPU or memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

### 3.2 Resource Monitoring Commands

```bash
# Get resource requests and limits for all pods
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
CPU_REQ:.spec.containers[*].resources.requests.cpu,\
MEM_REQ:.spec.containers[*].resources.requests.memory,\
CPU_LIM:.spec.containers[*].resources.limits.cpu,\
MEM_LIM:.spec.containers[*].resources.limits.memory

# Check namespace resource quotas
kubectl describe resourcequota -n <namespace>

# Check LimitRanges
kubectl describe limitrange -n <namespace>
```

### 3.3 Events Monitoring

```bash
# View cluster events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Watch events in real-time
kubectl get events -w

# Filter events by type
kubectl get events --field-selector type=Warning

# Events for specific resource
kubectl get events --field-selector involvedObject.name=<pod-name>
```

---

## 4. Manage and Evaluate Container Output Streams

### 4.1 Viewing Container Logs

```bash
# Basic log viewing
kubectl logs <pod-name>

# Logs from specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# Follow logs (streaming)
kubectl logs -f <pod-name>

# Show last N lines
kubectl logs --tail=100 <pod-name>

# Show logs since a time
kubectl logs --since=1h <pod-name>
kubectl logs --since-time=2024-01-01T10:00:00Z <pod-name>

# Previous container logs (after restart)
kubectl logs --previous <pod-name>

# Logs from all containers in pod
kubectl logs <pod-name> --all-containers=true

# Logs from all pods with a label
kubectl logs -l app=nginx --all-containers=true
```

### 4.2 Container Log Architecture

- Container logs are written to `/var/log/containers/` on the node
- Kubelet handles log rotation
- Default: 10MB per file, 5 files max
- Log format: `<pod-name>_<namespace>_<container-name>-<container-id>.log`

```bash
# View logs directly on node
ls -la /var/log/containers/
cat /var/log/containers/<pod-name>_<namespace>_<container-name>-<id>.log

# View kubelet logs
journalctl -u kubelet

# Check container runtime logs
journalctl -u containerd
```

### 4.3 Debugging with Exec and Ephemeral Containers

```bash
# Execute command in container
kubectl exec <pod-name> -- <command>
kubectl exec -it <pod-name> -- /bin/bash

# Execute in specific container
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Create debug/ephemeral container (K8s 1.25+)
kubectl debug <pod-name> -it --image=busybox --target=<container-name>

# Debug node
kubectl debug node/<node-name> -it --image=ubuntu
```

### 4.4 Common Log Patterns to Look For

| Pattern | Indicates |
|---------|-----------|
| `OOMKilled` | Container exceeded memory limit |
| `CrashLoopBackOff` | Container repeatedly crashing |
| `ImagePullBackOff` | Cannot pull container image |
| `ErrImagePull` | Image pull error |
| `CreateContainerError` | Failed to create container |
| `Error` or `FATAL` | Application errors |
| `connection refused` | Service connectivity issues |
| `permission denied` | RBAC or filesystem permissions |

---

## 5. Troubleshoot Services and Networking

### 5.1 Service Troubleshooting

```bash
# Check service details
kubectl get svc <service-name>
kubectl describe svc <service-name>

# Check endpoints
kubectl get endpoints <service-name>
kubectl get ep <service-name>

# Verify service selector matches pod labels
kubectl get pods --show-labels
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'

# Test service DNS resolution
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup <service-name>

# Test service connectivity
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- <service-name>:<port>
```

### 5.2 Common Service Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No endpoints | Selector doesn't match pod labels | Fix selector or pod labels |
| Service not resolving | CoreDNS issue | Check CoreDNS pods/logs |
| Connection refused | Pod not listening on port | Check targetPort matches container port |
| Timeout | NetworkPolicy blocking | Check NetworkPolicies |

### 5.3 DNS Troubleshooting

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS from a pod
kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default

# Check DNS configuration in pod
kubectl exec <pod-name> -- cat /etc/resolv.conf

# Expected resolv.conf content:
# nameserver <cluster-dns-ip>  (usually 10.96.0.10)
# search <namespace>.svc.cluster.local svc.cluster.local cluster.local
# ndots:5
```

### 5.4 Network Policy Troubleshooting

```bash
# List network policies
kubectl get networkpolicies --all-namespaces
kubectl get netpol -A

# Describe network policy
kubectl describe networkpolicy <policy-name>

# Check if network policy is blocking traffic:
# 1. Identify pods selected by the policy
# 2. Check ingress/egress rules
# 3. Test connectivity between pods

# Test pod-to-pod connectivity
kubectl exec <source-pod> -- curl <destination-pod-ip>:<port>
```

### 5.5 Pod Networking Troubleshooting

```bash
# Get pod IP
kubectl get pod <pod-name> -o wide

# Check pod network configuration
kubectl exec <pod-name> -- ip addr
kubectl exec <pod-name> -- ip route

# Check connectivity to another pod
kubectl exec <pod-name> -- ping <other-pod-ip>

# Check CNI plugin
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf

# Check CNI plugin logs (varies by plugin)
# For Calico:
kubectl logs -n kube-system -l k8s-app=calico-node
# For Flannel:
kubectl logs -n kube-system -l app=flannel
```

### 5.6 Ingress Troubleshooting

```bash
# Check ingress resources
kubectl get ingress --all-namespaces
kubectl describe ingress <ingress-name>

# Check ingress controller pods
kubectl get pods -n ingress-nginx  # or appropriate namespace

# Check ingress controller logs
kubectl logs -n ingress-nginx <ingress-controller-pod>

# Verify backend service exists and has endpoints
kubectl get svc <backend-service>
kubectl get endpoints <backend-service>
```

---

## 6. Pod Troubleshooting Flowchart

### 6.1 Pod Status Analysis

| Status | Meaning | Troubleshooting Steps |
|--------|---------|----------------------|
| `Pending` | Pod waiting to be scheduled | Check events, node resources, taints/tolerations |
| `ContainerCreating` | Container being set up | Check image pull, volume mounts |
| `Running` | Containers running | Check if app is healthy |
| `CrashLoopBackOff` | Container repeatedly crashing | Check logs, resource limits |
| `Error` | Container exited with error | Check logs, exec into container |
| `Completed` | Container finished successfully | Normal for Jobs |
| `Terminating` | Pod being deleted | Check finalizers if stuck |

### 6.2 Pod Troubleshooting Commands

```bash
# Full troubleshooting sequence
kubectl get pod <pod-name> -o wide          # Basic info
kubectl describe pod <pod-name>              # Detailed info + events
kubectl logs <pod-name>                      # Application logs
kubectl logs <pod-name> --previous           # Previous container logs
kubectl exec -it <pod-name> -- /bin/sh       # Interactive shell
kubectl get events --field-selector involvedObject.name=<pod-name>  # Events

# Check pod YAML for issues
kubectl get pod <pod-name> -o yaml
```

### 6.3 Common Pod Issues

#### ImagePullBackOff
```bash
# Check image name and tag
kubectl describe pod <pod-name> | grep Image

# Check imagePullSecrets
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'

# Verify secret exists
kubectl get secret <secret-name>
```

#### CrashLoopBackOff
```bash
# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Common causes:
# - Application error
# - Missing configuration
# - Resource limits too low
# - Liveness probe failing
```

#### Pod Stuck in Pending
```bash
kubectl describe pod <pod-name>

# Common causes:
# - Insufficient resources
# - Node selector/affinity not satisfied
# - Taints not tolerated
# - PVC not bound
```

---

## 7. Quick Troubleshooting Checklist

### Cluster Level
- [ ] All nodes Ready? `kubectl get nodes`
- [ ] Control plane pods running? `kubectl get pods -n kube-system`
- [ ] Can access API? `kubectl cluster-info`
- [ ] Events show issues? `kubectl get events --sort-by='.lastTimestamp'`

### Node Level
- [ ] Kubelet running? `systemctl status kubelet`
- [ ] Container runtime running? `systemctl status containerd`
- [ ] Sufficient resources? `kubectl top nodes`
- [ ] Kubelet logs clean? `journalctl -u kubelet`

### Application Level
- [ ] Pod running? `kubectl get pods`
- [ ] Container logs clean? `kubectl logs <pod>`
- [ ] Service has endpoints? `kubectl get endpoints`
- [ ] DNS working? `nslookup <service>`
- [ ] Network policies blocking? `kubectl get netpol`

---

## 8. Practice Exercises

### Exercise 1: Fix a Broken Kubelet

Scenario: A worker node shows NotReady status.

<details>
<summary>Troubleshooting Steps</summary>

```bash
# 1. SSH to the node
ssh <node>

# 2. Check kubelet status
systemctl status kubelet

# 3. Check kubelet logs
journalctl -u kubelet -n 50

# 4. Common fixes:
# - Restart kubelet
systemctl restart kubelet

# - Fix certificate issues
# - Correct configuration in /var/lib/kubelet/config.yaml
# - Restart container runtime
systemctl restart containerd
```

</details>

### Exercise 2: Debug Pod CrashLoopBackOff

Scenario: A pod is in CrashLoopBackOff state.

<details>
<summary>Troubleshooting Steps</summary>

```bash
# 1. Get pod status
kubectl describe pod <pod-name>

# 2. Check current and previous logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# 3. Check resource limits
kubectl get pod <pod-name> -o yaml | grep -A 10 resources

# 4. Common causes:
# - Application startup error
# - Missing ConfigMap/Secret
# - Memory limit too low (OOMKilled)
# - Liveness probe failing too quickly
```

</details>

### Exercise 3: Service Not Accessible

Scenario: A service exists but cannot be reached from other pods.

<details>
<summary>Troubleshooting Steps</summary>

```bash
# 1. Verify service exists and has correct selector
kubectl describe svc <service-name>

# 2. Check endpoints
kubectl get endpoints <service-name>
# If no endpoints, selector doesn't match any pods

# 3. Verify pod labels match service selector
kubectl get pods --show-labels

# 4. Test DNS resolution
kubectl run test --image=busybox:1.28 --rm -it -- nslookup <service-name>

# 5. Check NetworkPolicies
kubectl get networkpolicies

# 6. Test direct pod connectivity
kubectl exec <pod> -- curl <pod-ip>:<port>
```

</details>

---

## 9. Key Points for the Exam

1. **Always check events first**: `kubectl describe` and `kubectl get events`
2. **Know log locations**: Container logs, kubelet logs, control plane logs
3. **Understand the troubleshooting hierarchy**: Cluster → Node → Pod → Container
4. **Master kubectl commands**: `describe`, `logs`, `exec`, `get events`
5. **Know control plane component locations**: `/etc/kubernetes/manifests/`
6. **Understand DNS**: Service discovery, CoreDNS, resolv.conf
7. **Network troubleshooting**: Services, Endpoints, NetworkPolicies
8. **Resource monitoring**: `kubectl top`, Metrics Server

---

## 10. Documentation References

During the exam, you can access:
- https://kubernetes.io/docs/tasks/debug/
- https://kubernetes.io/docs/tasks/debug/debug-cluster/
- https://kubernetes.io/docs/tasks/debug/debug-application/
- https://kubernetes.io/docs/concepts/cluster-administration/logging/