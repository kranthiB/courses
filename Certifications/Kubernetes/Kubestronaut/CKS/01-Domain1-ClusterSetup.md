# CKS Domain 1: Cluster Setup (10%)

## Overview

This domain focuses on best practice configuration to control the environment's access, rights, and platform conformity. It covers network security policies, CIS benchmarks, ingress security, node metadata protection, and binary verification.

---

## 1.1 Use Network Security Policies to Restrict Cluster Level Access

### Concept Overview

Network Policies are Kubernetes resources that control traffic flow between pods and between pods and external endpoints. By default, all pods in a Kubernetes cluster can communicate with each other without restrictions. Network Policies allow you to implement microsegmentation.

### Key Points

- Network Policies are **namespaced** resources
- They require a **CNI plugin** that supports Network Policies (Calico, Cilium, Weave Net, etc.)
- Policies are **additive** - if any policy selects a pod, the pod is restricted and only allowed traffic explicitly permitted
- If no policy selects a pod, all traffic is allowed (default allow)
- An empty `podSelector: {}` selects all pods in the namespace

### Default Deny All Ingress Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # Selects all pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress
```

### Default Deny All Egress Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all egress
```

### Default Deny All Traffic (Ingress and Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow Specific Ingress Traffic

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
    ports:
    - protocol: TCP
      port: 8080
```

### Allow Egress to Specific CIDR

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-client
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
    ports:
    - protocol: TCP
      port: 443
```

### Block Access to Cloud Metadata API

This is crucial for preventing SSRF attacks that could expose cloud credentials:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cloud-metadata
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # AWS/GCP/Azure metadata endpoint
```

### Cross-Namespace Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
```

### Useful kubectl Commands

```bash
# List all network policies
kubectl get networkpolicies -A

# Describe a network policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Create network policy from file
kubectl apply -f network-policy.yaml

# Delete network policy
kubectl delete networkpolicy <policy-name> -n <namespace>

# Test connectivity (from inside a pod)
kubectl exec -it <pod-name> -- wget -qO- --timeout=2 http://<target-service>
kubectl exec -it <pod-name> -- nc -zv <target-ip> <port>
```

---

## 1.2 Use CIS Benchmark to Review Security Configuration

### What is CIS Benchmark?

The Center for Internet Security (CIS) Kubernetes Benchmark provides prescriptive guidance for establishing a secure configuration posture for Kubernetes. It covers security recommendations for:

- Control plane components (API server, etcd, scheduler, controller-manager)
- Worker node components (kubelet, kube-proxy)
- Policies (RBAC, Pod Security, Network Policies, Secrets)

### Kube-bench

**Kube-bench** is an open-source tool by Aqua Security that checks whether Kubernetes is deployed securely by running the checks documented in the CIS Kubernetes Benchmark.

#### Installation

```bash
# Download and install kube-bench
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.7.0/kube-bench_0.7.0_linux_amd64.tar.gz -o kube-bench.tar.gz
tar -xvf kube-bench.tar.gz

# Or run as a container
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

#### Running Kube-bench

```bash
# Run all checks on a master node
./kube-bench run --targets master

# Run all checks on a worker node
./kube-bench run --targets node

# Run specific checks
./kube-bench run --targets master --check 1.1.1,1.1.2

# Output to JSON
./kube-bench run --json > results.json

# Run as a Kubernetes Job
kubectl apply -f job.yaml
kubectl logs job/kube-bench
```

#### Sample Output Analysis

```
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 644 or more restrictive
[FAIL] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root
[WARN] 1.1.3 Ensure that the controller manager pod specification file permissions are set to 644 or more restrictive
```

### Key CIS Benchmark Checks

#### API Server Security

```bash
# Check API server configuration
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Important flags to verify:
# --anonymous-auth=false
# --authorization-mode=Node,RBAC
# --enable-admission-plugins=NodeRestriction,PodSecurityAdmission
# --audit-log-path=/var/log/audit.log
# --audit-log-maxage=30
# --audit-log-maxbackup=3
# --audit-log-maxsize=100
# --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
# --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
# --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

#### etcd Security

```bash
# Check etcd configuration
cat /etc/kubernetes/manifests/etcd.yaml

# Important flags:
# --cert-file=/etc/kubernetes/pki/etcd/server.crt
# --key-file=/etc/kubernetes/pki/etcd/server.key
# --client-cert-auth=true
# --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
# --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
# --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
# --peer-client-cert-auth=true
# --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

#### Kubelet Security

```bash
# Check kubelet configuration
cat /var/lib/kubelet/config.yaml

# Important settings:
# authentication:
#   anonymous:
#     enabled: false
#   webhook:
#     enabled: true
# authorization:
#   mode: Webhook
# readOnlyPort: 0
# protectKernelDefaults: true
```

### File Permission Checks

```bash
# Control plane node file permissions
stat -c %a /etc/kubernetes/manifests/kube-apiserver.yaml    # Should be 644 or more restrictive
stat -c %a /etc/kubernetes/manifests/kube-controller-manager.yaml
stat -c %a /etc/kubernetes/manifests/kube-scheduler.yaml
stat -c %a /etc/kubernetes/manifests/etcd.yaml
stat -c %a /etc/kubernetes/admin.conf                        # Should be 600

# Check ownership (should be root:root)
ls -la /etc/kubernetes/manifests/
ls -la /etc/kubernetes/pki/
```

---

## 1.3 Properly Set Up Ingress Objects with Security Control

### Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Creating TLS Secret

```bash
# Generate self-signed certificate (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.example.com/O=MyOrg"

# Create TLS secret
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n production
```

### Ingress Security Annotations (NGINX)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    # Force HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Enable HSTS
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    
    # IP Whitelisting
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
    
    # Client certificate authentication
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## 1.4 Protect Node Metadata and Endpoints

### The Metadata API Threat

Cloud providers expose instance metadata APIs (usually at 169.254.169.254) that can reveal sensitive information including:
- Instance credentials and IAM roles
- User data scripts (may contain secrets)
- Network configuration
- Instance identity documents

### Network Policy to Block Metadata Access

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata-access
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```

### Kubelet Security Configuration

The kubelet API should be secured:

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 2m0s
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
readOnlyPort: 0  # Disable read-only port
```

### Verify Kubelet Security

```bash
# Check if anonymous auth is disabled
curl -sk https://localhost:10250/pods
# Should return 401 Unauthorized

# Check if read-only port is disabled
curl -s http://localhost:10255/pods
# Should fail to connect
```

---

## 1.5 Minimize Access to GUI Elements

### Kubernetes Dashboard Security

If dashboard is required, secure it properly:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin  # Be careful with this!
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
```

### Best Practices

1. **Don't expose dashboard publicly** - Use kubectl proxy or port-forward
2. **Use proper authentication** - Never use skip login
3. **Apply RBAC** - Use minimal required permissions
4. **Use Network Policies** - Restrict access to dashboard service

```bash
# Secure access via kubectl proxy
kubectl proxy

# Access at: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

---

## 1.6 Verify Platform Binaries Before Deploying

### Verifying Kubernetes Binaries

```bash
# Download binary and checksum
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl.sha256"

# Verify the binary
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
# Expected output: kubectl: OK

# Make executable and move
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Verifying kubeadm and kubelet

```bash
# Download and verify kubeadm
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubeadm"
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubeadm.sha256"
echo "$(cat kubeadm.sha256)  kubeadm" | sha256sum --check

# Download and verify kubelet
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubelet"
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubelet.sha256"
echo "$(cat kubelet.sha256)  kubelet" | sha256sum --check
```

### Verifying Running Binaries

```bash
# Get the hash of running kubelet
sha256sum /usr/bin/kubelet

# Compare with official hash from:
# https://github.com/kubernetes/kubernetes/releases

# Check version
kubelet --version
kubectl version --client
kubeadm version
```

---

## Hands-On Lab Exercises

### Lab 1: Network Policy Implementation

```bash
# Create test namespace
kubectl create namespace np-test

# Deploy test pods
kubectl run frontend --image=nginx -n np-test -l app=frontend
kubectl run backend --image=nginx -n np-test -l app=backend
kubectl run database --image=nginx -n np-test -l app=database

# Expose services
kubectl expose pod frontend --port=80 -n np-test
kubectl expose pod backend --port=80 -n np-test
kubectl expose pod database --port=80 -n np-test

# Test connectivity (should work initially)
kubectl exec frontend -n np-test -- curl -s backend.np-test
kubectl exec backend -n np-test -- curl -s database.np-test

# Apply default deny
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: np-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Test connectivity (should fail now)
kubectl exec frontend -n np-test -- curl -s --max-time 3 backend.np-test

# Allow frontend -> backend
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: np-test
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 80
EOF

# Also allow egress from frontend
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: np-test
spec:
  podSelector:
    matchLabels:
      app: frontend
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - port: 80
EOF
```

### Lab 2: Run Kube-bench

```bash
# Run kube-bench as a Job
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Wait for completion
kubectl wait --for=condition=complete job/kube-bench -n default --timeout=300s

# View results
kubectl logs job/kube-bench

# Cleanup
kubectl delete job kube-bench
```

---

## Practice Questions

1. **Create a Network Policy** that allows pods with label `role=db` to only receive traffic from pods with label `role=api` on port 3306.

2. **Write a Network Policy** that blocks all egress traffic except to DNS (port 53) and the service CIDR 10.96.0.0/12.

3. **What kube-bench command** would you run to check only the kubelet configuration?

4. **Create an Ingress resource** with TLS termination for domain `secure.example.com`.

5. **How do you verify** that the kubectl binary you downloaded is authentic?

---

## Quick Reference

### Network Policy Selectors

| Selector | Purpose |
|----------|---------|
| `podSelector` | Select pods by labels |
| `namespaceSelector` | Select namespaces by labels |
| `ipBlock` | Select IP ranges (CIDR) |

### CIS Benchmark Categories

| ID | Category |
|----|----------|
| 1.x | Control Plane Components |
| 2.x | etcd |
| 3.x | Control Plane Configuration |
| 4.x | Worker Nodes |
| 5.x | Policies |

### Essential File Locations

| File | Purpose |
|------|---------|
| `/etc/kubernetes/manifests/` | Static pod manifests |
| `/etc/kubernetes/pki/` | Certificates |
| `/var/lib/kubelet/config.yaml` | Kubelet configuration |
| `/etc/kubernetes/admin.conf` | Admin kubeconfig |

---

## Official Documentation References

*These URLs are accessible during the CKS exam:*

- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [Ingress TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)
- [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [Kubelet Configuration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)