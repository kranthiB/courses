# KCSA Hands-On Labs - Complete Practical Guide

## Lab Environment Setup

### Option 1: Minikube (Recommended)
```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster with PSA enabled
minikube start --kubernetes-version=v1.28.0

# Verify
kubectl cluster-info
kubectl get nodes
```

### Option 2: kind (Kubernetes in Docker)
```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name security-lab

# Verify
kubectl cluster-info
```

---

## LAB 1: Pod Security Standards (PSS) & Pod Security Admission (PSA)

### Objective
Configure namespace-level security enforcement using Pod Security Admission.

### Step 1: Create Test Namespaces
```bash
# Create namespaces for each security level
kubectl create namespace pss-privileged
kubectl create namespace pss-baseline
kubectl create namespace pss-restricted
```

### Step 2: Apply PSA Labels
```bash
# Privileged namespace (allows everything)
kubectl label namespace pss-privileged \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged

# Baseline namespace (minimal restrictions)
kubectl label namespace pss-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/audit=baseline

# Restricted namespace (hardened)
kubectl label namespace pss-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

### Step 3: Verify Labels
```bash
kubectl get namespace pss-privileged --show-labels
kubectl get namespace pss-baseline --show-labels
kubectl get namespace pss-restricted --show-labels
```

### Step 4: Test with Privileged Pod
```bash
# Create privileged pod spec
cat <<EOF > privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      privileged: true
EOF

# Try in each namespace
kubectl apply -f privileged-pod.yaml -n pss-privileged  # SUCCESS
kubectl apply -f privileged-pod.yaml -n pss-baseline    # FAIL
kubectl apply -f privileged-pod.yaml -n pss-restricted  # FAIL
```

### Step 5: Test with Baseline-Compliant Pod
```bash
cat <<EOF > baseline-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: true  # Allowed in baseline
EOF

kubectl apply -f baseline-pod.yaml -n pss-baseline    # SUCCESS
kubectl apply -f baseline-pod.yaml -n pss-restricted  # FAIL
```

### Step 6: Test with Restricted-Compliant Pod
```bash
cat <<EOF > restricted-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
EOF

kubectl apply -f restricted-pod.yaml -n pss-restricted  # SUCCESS
```

### Step 7: Clean Up
```bash
kubectl delete namespace pss-privileged pss-baseline pss-restricted
```

### ✅ Lab 1 Checkpoint
- [ ] Understand the three PSS levels
- [ ] Can apply PSA labels to namespaces
- [ ] Know what configurations fail at each level

---

## LAB 2: RBAC Configuration

### Objective
Create and test Role-Based Access Control configurations.

### Step 1: Create Namespace and Service Account
```bash
kubectl create namespace rbac-lab
kubectl create serviceaccount developer -n rbac-lab
```

### Step 2: Create a Role (Namespace-Scoped)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-lab
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF
```

### Step 3: Create RoleBinding
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: rbac-lab
subjects:
- kind: ServiceAccount
  name: developer
  namespace: rbac-lab
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Step 4: Test Permissions
```bash
# Check what developer can do
kubectl auth can-i get pods -n rbac-lab --as=system:serviceaccount:rbac-lab:developer
# Output: yes

kubectl auth can-i create pods -n rbac-lab --as=system:serviceaccount:rbac-lab:developer
# Output: no

kubectl auth can-i delete pods -n rbac-lab --as=system:serviceaccount:rbac-lab:developer
# Output: no

kubectl auth can-i get secrets -n rbac-lab --as=system:serviceaccount:rbac-lab:developer
# Output: no

# List all permissions
kubectl auth can-i --list --as=system:serviceaccount:rbac-lab:developer -n rbac-lab
```

### Step 5: Create ClusterRole and ClusterRoleBinding
```bash
# ClusterRole for viewing nodes (cluster-wide)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
EOF

# ClusterRoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-nodes
subjects:
- kind: ServiceAccount
  name: developer
  namespace: rbac-lab
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
EOF

# Test
kubectl auth can-i get nodes --as=system:serviceaccount:rbac-lab:developer
# Output: yes
```

### Step 6: Create Developer Role with More Permissions
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: rbac-lab
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Read-only for secrets
EOF
```

### Step 7: Clean Up
```bash
kubectl delete namespace rbac-lab
kubectl delete clusterrole node-viewer
kubectl delete clusterrolebinding view-nodes
```

### ✅ Lab 2 Checkpoint
- [ ] Can create Role and ClusterRole
- [ ] Understand RoleBinding vs ClusterRoleBinding
- [ ] Can test permissions with `kubectl auth can-i`

---

## LAB 3: Network Policies

### Objective
Implement network segmentation using Network Policies.

### Step 1: Create Test Namespace and Pods
```bash
kubectl create namespace netpol-lab

# Create a web pod
kubectl run web --image=nginx --port=80 -n netpol-lab --labels="app=web"
kubectl expose pod web --port=80 -n netpol-lab

# Create a database pod
kubectl run db --image=redis --port=6379 -n netpol-lab --labels="app=db"
kubectl expose pod db --port=6379 -n netpol-lab

# Create a test pod for connectivity testing
kubectl run test --image=busybox -n netpol-lab --labels="app=test" -- sleep 3600
```

### Step 2: Test Default Connectivity (Allow All)
```bash
# Wait for pods to be ready
kubectl wait --for=condition=Ready pod/web pod/db pod/test -n netpol-lab --timeout=60s

# Test connectivity from test pod to web
kubectl exec -n netpol-lab test -- wget -qO- --timeout=2 http://web
# Should work

# Test connectivity from test pod to db
kubectl exec -n netpol-lab test -- nc -zv db 6379
# Should work
```

### Step 3: Apply Default Deny All Ingress
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: netpol-lab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Test connectivity again
kubectl exec -n netpol-lab test -- wget -qO- --timeout=2 http://web
# Should FAIL (timeout)
```

### Step 4: Allow Traffic to Web Pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: test
    ports:
    - protocol: TCP
      port: 80
EOF

# Test connectivity
kubectl exec -n netpol-lab test -- wget -qO- --timeout=2 http://web
# Should work now
```

### Step 5: Allow Only Web to Access DB
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-from-web
  namespace: netpol-lab
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
          app: web
    ports:
    - protocol: TCP
      port: 6379
EOF

# Test from test pod (should fail)
kubectl exec -n netpol-lab test -- nc -zv db 6379
# Should FAIL

# Test from web pod (should work)
kubectl exec -n netpol-lab web -- apt-get update && apt-get install -y netcat-openbsd
kubectl exec -n netpol-lab web -- nc -zv db 6379
# Should work
```

### Step 6: Deny All Egress
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: netpol-lab
spec:
  podSelector: {}
  policyTypes:
  - Egress
EOF

# Allow DNS egress (required for service discovery)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: netpol-lab
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
EOF
```

### Step 7: View Network Policies
```bash
kubectl get networkpolicies -n netpol-lab
kubectl describe networkpolicy allow-web-ingress -n netpol-lab
```

### Step 8: Clean Up
```bash
kubectl delete namespace netpol-lab
```

### ✅ Lab 3 Checkpoint
- [ ] Understand default allow-all behavior
- [ ] Can create deny-all policies
- [ ] Can allow specific traffic patterns
- [ ] Know DNS egress is needed for service discovery

---

## LAB 4: Secrets Management

### Objective
Create, use, and secure Kubernetes Secrets.

### Step 1: Create Secrets (Multiple Methods)
```bash
kubectl create namespace secrets-lab

# Method 1: From literal values
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=S3cr3tP@ss \
  -n secrets-lab

# Method 2: From file
echo -n 'my-api-key-12345' > api-key.txt
kubectl create secret generic api-secret \
  --from-file=api-key=api-key.txt \
  -n secrets-lab
rm api-key.txt

# Method 3: From YAML (base64 encoded)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: secrets-lab
type: Opaque
data:
  tls.crt: $(echo -n 'cert-content' | base64)
  tls.key: $(echo -n 'key-content' | base64)
EOF
```

### Step 2: View Secrets (Understanding Base64)
```bash
# View secret
kubectl get secret db-creds -n secrets-lab -o yaml

# Decode secret
kubectl get secret db-creds -n secrets-lab -o jsonpath='{.data.password}' | base64 -d
echo  # newline

# IMPORTANT: This shows secrets are NOT encrypted, just encoded!
```

### Step 3: Use Secret as Environment Variables
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
  namespace: secrets-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo Username: \$DB_USER Password: \$DB_PASS && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
EOF

# View the output
kubectl logs secret-env-pod -n secrets-lab
```

### Step 4: Use Secret as Volume Mount
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
  namespace: secrets-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /etc/secrets/username && echo && cat /etc/secrets/password && sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-creds
EOF

# View the output
kubectl logs secret-vol-pod -n secrets-lab
```

### Step 5: Restrict Secret Access with RBAC
```bash
# Create a service account that can only read specific secrets
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: limited-sa
  namespace: secrets-lab
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: secrets-lab
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-creds"]  # Only this specific secret
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-db-creds
  namespace: secrets-lab
subjects:
- kind: ServiceAccount
  name: limited-sa
  namespace: secrets-lab
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Test
kubectl auth can-i get secret/db-creds -n secrets-lab --as=system:serviceaccount:secrets-lab:limited-sa
# yes

kubectl auth can-i get secret/api-secret -n secrets-lab --as=system:serviceaccount:secrets-lab:limited-sa
# no
```

### Step 6: Disable Auto-Mount of Service Account Token
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
  namespace: secrets-lab
spec:
  automountServiceAccountToken: false
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
EOF

# Verify no token mounted
kubectl exec no-token-pod -n secrets-lab -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# Should fail or be empty
```

### Step 7: Clean Up
```bash
kubectl delete namespace secrets-lab
```

### ✅ Lab 4 Checkpoint
- [ ] Know secrets are base64 encoded, NOT encrypted
- [ ] Can create secrets multiple ways
- [ ] Can use secrets as env vars and volumes
- [ ] Understand RBAC restrictions on secrets

---

## LAB 5: Security Context Configuration

### Objective
Configure pod and container security contexts.

### Step 1: Create Namespace
```bash
kubectl create namespace secctx-lab
```

### Step 2: Run as Non-Root User
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-pod
  namespace: secctx-lab
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "id && sleep 3600"]
EOF

kubectl logs nonroot-pod -n secctx-lab
# Output: uid=1000 gid=3000 groups=2000
```

### Step 3: Read-Only Root Filesystem
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: readonly-pod
  namespace: secctx-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "touch /test.txt 2>&1 || echo 'Write failed as expected' && sleep 3600"]
    securityContext:
      readOnlyRootFilesystem: true
EOF

kubectl logs readonly-pod -n secctx-lab
# Should show: Read-only file system or Write failed
```

### Step 4: Drop All Capabilities
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-caps-pod
  namespace: secctx-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /proc/1/status | grep Cap && sleep 3600"]
    securityContext:
      capabilities:
        drop:
        - ALL
EOF

kubectl logs no-caps-pod -n secctx-lab
# CapEff should be 0000000000000000
```

### Step 5: Add Specific Capability
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: net-bind-pod
  namespace: secctx-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
EOF
```

### Step 6: Prevent Privilege Escalation
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-privesc-pod
  namespace: secctx-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
EOF
```

### Step 7: Complete Secure Pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secctx-lab
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
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
EOF

kubectl get pod secure-pod -n secctx-lab -o yaml | grep -A 30 securityContext
```

### Step 8: Clean Up
```bash
kubectl delete namespace secctx-lab
```

### ✅ Lab 5 Checkpoint
- [ ] Can configure runAsNonRoot and runAsUser
- [ ] Understand readOnlyRootFilesystem
- [ ] Know how to drop/add capabilities
- [ ] Can configure complete secure pod

---

## LAB 6: Security Scanning Tools

### Objective
Use kube-bench, Kubescape, and Trivy for security scanning.

### Step 1: Run kube-bench
```bash
# Run as a Job
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Wait for completion
kubectl wait --for=condition=complete job/kube-bench --timeout=120s

# View results
kubectl logs job/kube-bench

# Clean up
kubectl delete job kube-bench
```

### Step 2: Install and Run Kubescape
```bash
# Install Kubescape
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | bash

# Scan with NSA framework
kubescape scan framework nsa

# Scan with CIS benchmark
kubescape scan framework cis-v1.23-t1.0.1

# Scan specific namespace
kubescape scan framework nsa --include-namespaces default

# Scan YAML file
kubescape scan deployment.yaml
```

### Step 3: Install and Run Trivy
```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Scan an image
trivy image nginx:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan Kubernetes manifests
trivy config ./

# Scan running cluster
trivy k8s cluster --report summary
```

### Step 4: Create Vulnerable Pod and Scan
```bash
cat <<EOF > vulnerable-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      privileged: true
      runAsUser: 0
EOF

# Scan with Kubescape
kubescape scan vulnerable-pod.yaml

# Scan with Trivy
trivy config vulnerable-pod.yaml
```

### ✅ Lab 6 Checkpoint
- [ ] Can run kube-bench
- [ ] Can use Kubescape with different frameworks
- [ ] Can scan images and configs with Trivy

---

## LAB 7: Admission Control with Kyverno

### Objective
Implement policy enforcement using Kyverno.

### Step 1: Install Kyverno
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml

# Wait for Kyverno to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=kyverno -n kyverno --timeout=120s
```

### Step 2: Create Policy - Require Labels
```bash
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-team-label
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Label 'team' is required"
      pattern:
        metadata:
          labels:
            team: "?*"
EOF

# Test - should fail
kubectl run test-pod --image=nginx
# Error: Label 'team' is required

# Test - should succeed
kubectl run test-pod --image=nginx --labels="team=backend"
kubectl delete pod test-pod
```

### Step 3: Create Policy - Disallow Privileged
```bash
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
  - name: deny-privileged
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Privileged containers are not allowed"
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: "!true"
EOF
```

### Step 4: Create Mutating Policy - Add Default Labels
```bash
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
  - name: add-managed-by
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            managed-by: kyverno
EOF

# Test
kubectl run mutation-test --image=nginx --labels="team=test"
kubectl get pod mutation-test --show-labels
# Should have managed-by=kyverno label
kubectl delete pod mutation-test
```

### Step 5: Clean Up
```bash
kubectl delete clusterpolicy require-labels disallow-privileged add-default-labels
kubectl delete -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml
```

### ✅ Lab 7 Checkpoint
- [ ] Can install Kyverno
- [ ] Can create validating policies
- [ ] Can create mutating policies
- [ ] Understand Enforce vs Audit modes

---

## LAB 8: Audit Logging

### Objective
Configure and analyze Kubernetes audit logs.

### Step 1: Create Audit Policy
```bash
cat <<EOF > audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all requests to secrets at RequestResponse level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log pod changes at Request level
- level: Request
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["create", "update", "patch", "delete"]

# Log authentication failures
- level: Metadata
  nonResourceURLs:
  - "/api*"
  - "/version"

# Don't log read-only requests to certain resources
- level: None
  resources:
  - group: ""
    resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

# Default: log at Metadata level
- level: Metadata
EOF
```

### Step 2: Configure API Server (Minikube)
```bash
# For Minikube, you need to configure the API server
minikube ssh
sudo mkdir -p /var/log/kubernetes/audit
exit

# Start minikube with audit logging (requires restart)
minikube stop
minikube start \
  --extra-config=apiserver.audit-policy-file=/etc/kubernetes/audit-policy.yaml \
  --extra-config=apiserver.audit-log-path=/var/log/kubernetes/audit/audit.log \
  --extra-config=apiserver.audit-log-maxage=30 \
  --extra-config=apiserver.audit-log-maxbackup=10 \
  --extra-config=apiserver.audit-log-maxsize=100
```

### Step 3: Generate Audit Events
```bash
# Create a secret (should be logged at RequestResponse)
kubectl create secret generic audit-test --from-literal=key=value

# Create a pod (should be logged at Request)
kubectl run audit-pod --image=nginx

# Delete them
kubectl delete secret audit-test
kubectl delete pod audit-pod
```

### Step 4: View Audit Logs
```bash
minikube ssh
sudo cat /var/log/kubernetes/audit/audit.log | jq .
exit
```

### ✅ Lab 8 Checkpoint
- [ ] Understand audit policy levels (None, Metadata, Request, RequestResponse)
- [ ] Can create audit policy
- [ ] Know how to analyze audit logs

---

## Summary: What You Should Be Able to Do

After completing all labs, you should be able to:

1. ✅ Configure Pod Security Admission at namespace level
2. ✅ Create and test RBAC roles and bindings
3. ✅ Implement network policies for segmentation
4. ✅ Manage secrets securely
5. ✅ Configure security contexts for pods
6. ✅ Run security scanning tools (kube-bench, Kubescape, Trivy)
7. ✅ Implement policies with Kyverno
8. ✅ Configure and analyze audit logs

---

## Quick Commands Reference

```bash
# PSA Labels
kubectl label namespace <ns> pod-security.kubernetes.io/enforce=restricted

# RBAC Testing
kubectl auth can-i <verb> <resource> --as=<user>
kubectl auth can-i --list --as=<user>

# Network Policy
kubectl get networkpolicies -A
kubectl describe networkpolicy <name>

# Secrets
kubectl create secret generic <name> --from-literal=key=value
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d

# Security Context
kubectl get pod <name> -o jsonpath='{.spec.securityContext}'

# Scanning
kube-bench run
kubescape scan framework nsa
trivy image <image>
trivy config <path>
```