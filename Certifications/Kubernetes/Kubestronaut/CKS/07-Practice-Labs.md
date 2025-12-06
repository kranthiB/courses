# CKS Hands-On Practice Labs

## Introduction

These labs are designed to build muscle memory and practical skills. Complete each lab multiple times until you can finish them without referring to documentation.

**Target**: Complete each lab in the time specified. Repeat until you achieve this consistently.

---

## Lab Set 1: Network Policies (Target: 15 minutes for all)

### Lab 1.1: Default Deny All Traffic

**Scenario**: Secure the `production` namespace by denying all ingress and egress traffic by default.

```bash
# Setup
kubectl create namespace production
kubectl run web --image=nginx -n production -l app=web
kubectl run api --image=nginx -n production -l app=api
kubectl run db --image=nginx -n production -l app=db
kubectl expose pod web --port=80 -n production
kubectl expose pod api --port=80 -n production
kubectl expose pod db --port=80 -n production

# Wait for pods
kubectl wait --for=condition=Ready pod --all -n production --timeout=60s
```

**Task**: Create a NetworkPolicy that denies all ingress and egress traffic in the `production` namespace.

<details>
<summary>Solution</summary>

```bash
kubectl apply -f - <<EOF
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
EOF
```

**Verify**:
```bash
# This should timeout/fail
kubectl exec -n production web -- curl -s --max-time 3 api.production
```
</details>

---

### Lab 1.2: Allow Specific Traffic Flow

**Scenario**: Allow traffic flow: web → api → db (port 80)

**Task**: Create NetworkPolicies to allow:
1. `web` pods to communicate with `api` pods on port 80
2. `api` pods to communicate with `db` pods on port 80
3. Allow DNS resolution for all pods

<details>
<summary>Solution</summary>

```bash
# Allow DNS for all pods
kubectl apply -f - <<EOF
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
EOF

# Allow web -> api
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-from-web
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - port: 80
EOF

# Allow api -> db
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-from-api
  namespace: production
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
          app: api
    ports:
    - port: 80
EOF
```

**Verify**:
```bash
# web -> api should work
kubectl exec -n production web -- curl -s --max-time 3 api.production

# web -> db should fail
kubectl exec -n production web -- curl -s --max-time 3 db.production

# api -> db should work
kubectl exec -n production api -- curl -s --max-time 3 db.production
```
</details>

---

### Lab 1.3: Block Metadata API Access

**Scenario**: Prevent pods from accessing the cloud metadata API (169.254.169.254).

**Task**: Create a NetworkPolicy in namespace `secure-apps` that allows all egress EXCEPT to 169.254.169.254.

```bash
# Setup
kubectl create namespace secure-apps
kubectl run test-pod --image=nginx -n secure-apps
```

<details>
<summary>Solution</summary>

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata
  namespace: secure-apps
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
EOF
```
</details>

---

## Lab Set 2: RBAC (Target: 20 minutes for all)

### Lab 2.1: Create Developer Role

**Scenario**: Create RBAC for a developer who needs to:
- View, create, and delete pods in `dev` namespace
- View logs
- Cannot access secrets

```bash
# Setup
kubectl create namespace dev
```

**Task**: Create Role and RoleBinding for user `developer`.

<details>
<summary>Solution</summary>

```bash
# Create Role
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: developer-role
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
  name: developer-binding
  namespace: dev
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Verify**:
```bash
kubectl auth can-i get pods -n dev --as developer        # yes
kubectl auth can-i get pods/log -n dev --as developer    # yes
kubectl auth can-i get secrets -n dev --as developer     # no
kubectl auth can-i get pods -n default --as developer    # no
```
</details>

---

### Lab 2.2: Service Account with Minimal Permissions

**Scenario**: Create a ServiceAccount for an application that only needs to read ConfigMaps named `app-config`.

```bash
kubectl create namespace app-ns
kubectl create configmap app-config -n app-ns --from-literal=key=value
```

**Task**: 
1. Create ServiceAccount `app-sa` with `automountServiceAccountToken: false`
2. Create Role allowing only `get` on ConfigMap `app-config`
3. Create RoleBinding
4. Create a Pod using this ServiceAccount

<details>
<summary>Solution</summary>

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: app-ns
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: app-ns
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-configmap-reader
  namespace: app-ns
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: app-ns
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: app-ns
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
  containers:
  - name: app
    image: nginx
EOF
```

**Verify**:
```bash
kubectl auth can-i get configmap/app-config -n app-ns --as system:serviceaccount:app-ns:app-sa  # yes
kubectl auth can-i get configmap/other -n app-ns --as system:serviceaccount:app-ns:app-sa       # no
kubectl auth can-i list configmaps -n app-ns --as system:serviceaccount:app-ns:app-sa           # no
```
</details>

---

### Lab 2.3: Investigate Excessive Permissions

**Scenario**: Find and fix overly permissive RBAC.

```bash
# Setup - Create overly permissive setup
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: suspicious-binding
subjects:
- kind: User
  name: intern
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Task**: 
1. Find ClusterRoleBindings that grant `cluster-admin`
2. Remove the overly permissive binding
3. Create appropriate limited access for user `intern` (view-only in `intern-ns` namespace)

<details>
<summary>Solution</summary>

```bash
# Find cluster-admin bindings
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# Delete suspicious binding
kubectl delete clusterrolebinding suspicious-binding

# Create appropriate access
kubectl create namespace intern-ns

kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: intern-view
  namespace: intern-ns
subjects:
- kind: User
  name: intern
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Verify**:
```bash
kubectl auth can-i --list --as intern
kubectl auth can-i get pods -n intern-ns --as intern     # yes
kubectl auth can-i delete pods -n intern-ns --as intern  # no
kubectl auth can-i get pods -n kube-system --as intern   # no
```
</details>

---

## Lab Set 3: Pod Security (Target: 20 minutes for all)

### Lab 3.1: Enforce Restricted Pod Security Standard

**Scenario**: Configure namespace `secure-ns` to enforce the `restricted` Pod Security Standard.

**Task**:
1. Create namespace with appropriate labels
2. Try to create a non-compliant pod (should fail)
3. Create a compliant pod (should succeed)

<details>
<summary>Solution</summary>

```bash
# Create namespace with PSS labels
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
EOF

# Try non-compliant pod (will fail)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: secure-ns
spec:
  containers:
  - name: nginx
    image: nginx
EOF
# Should be rejected

# Create compliant pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: secure-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
EOF
```
</details>

---

### Lab 3.2: Create Immutable Container

**Scenario**: Create a pod with maximum security hardening.

**Task**: Create pod `hardened-pod` in namespace `hardened` with:
- Read-only root filesystem
- Run as non-root user (UID 10000)
- No privilege escalation
- All capabilities dropped
- Seccomp RuntimeDefault profile
- Resource limits

<details>
<summary>Solution</summary>

```bash
kubectl create namespace hardened

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  namespace: hardened
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10000
    runAsGroup: 10000
    fsGroup: 10000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: "200m"
        memory: "128Mi"
      requests:
        cpu: "100m"
        memory: "64Mi"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir:
      sizeLimit: "64Mi"
  - name: cache
    emptyDir:
      sizeLimit: "64Mi"
  - name: run
    emptyDir:
      sizeLimit: "1Mi"
EOF
```

**Verify**:
```bash
kubectl exec -n hardened hardened-pod -- id
# Should show uid=10000

kubectl exec -n hardened hardened-pod -- touch /test
# Should fail: Read-only file system
```
</details>

---

### Lab 3.3: Fix Insecure Pod

**Scenario**: An insecure pod exists. Fix its security issues.

```bash
# Setup - Create insecure pod
kubectl create namespace fix-me
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  namespace: fix-me
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true
      runAsUser: 0
      capabilities:
        add:
        - SYS_ADMIN
        - NET_ADMIN
EOF
```

**Task**: Delete the insecure pod and recreate it with proper security settings.

<details>
<summary>Solution</summary>

```bash
kubectl delete pod insecure-pod -n fix-me

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  namespace: fix-me
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: false
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx  
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
EOF
```
</details>

---

## Lab Set 4: Secrets Management (Target: 15 minutes for all)

### Lab 4.1: Create and Use Secrets

**Scenario**: Create secrets and mount them securely.

**Task**:
1. Create a secret `db-creds` with `username=admin` and `password=S3cr3tP@ss`
2. Create a pod that mounts this secret as a volume at `/etc/secrets` with mode 0400
3. The secret should NOT be exposed as environment variables

<details>
<summary>Solution</summary>

```bash
kubectl create namespace secrets-lab

# Create secret
kubectl create secret generic db-creds \
  -n secrets-lab \
  --from-literal=username=admin \
  --from-literal=password='S3cr3tP@ss'

# Create pod with secret volume
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
  namespace: secrets-lab
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-creds
      defaultMode: 0400
EOF
```

**Verify**:
```bash
kubectl exec -n secrets-lab secret-pod -- ls -la /etc/secrets
kubectl exec -n secrets-lab secret-pod -- cat /etc/secrets/username
kubectl exec -n secrets-lab secret-pod -- cat /etc/secrets/password
```
</details>

---

### Lab 4.2: Configure Encryption at Rest

**Scenario**: Enable encryption at rest for secrets (on control plane node).

**Task**:
1. Create EncryptionConfiguration
2. Configure API server to use it
3. Verify encryption works

<details>
<summary>Solution</summary>

```bash
# Generate encryption key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# Create encryption config directory
mkdir -p /etc/kubernetes/enc

# Create encryption config
cat > /etc/kubernetes/enc/enc.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

# Set proper permissions
chmod 600 /etc/kubernetes/enc/enc.yaml

# Edit kube-apiserver.yaml to add:
# - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
# And mount the enc directory

# After API server restarts, create a test secret
kubectl create secret generic encryption-test --from-literal=mykey=mydata

# Verify in etcd (data should be encrypted)
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/encryption-test | hexdump -C
```
</details>

---

## Lab Set 5: AppArmor and Seccomp (Target: 20 minutes for all)

### Lab 5.1: Apply AppArmor Profile

**Scenario**: Create and apply an AppArmor profile that denies file writes.

**Task** (on a node with AppArmor):
1. Create AppArmor profile `k8s-deny-write`
2. Load the profile
3. Create a pod using this profile
4. Verify writes are blocked

<details>
<summary>Solution</summary>

```bash
# On the node, create profile
cat > /etc/apparmor.d/k8s-deny-write <<EOF
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>
  
  file,
  
  deny /** w,
}
EOF

# Load profile
apparmor_parser -r /etc/apparmor.d/k8s-deny-write

# Verify profile is loaded
aa-status | grep k8s-deny-write

# Create pod with AppArmor profile
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/secure: localhost/k8s-deny-write
spec:
  containers:
  - name: secure
    image: nginx
EOF
```

**Verify**:
```bash
kubectl exec apparmor-pod -- touch /tmp/test
# Should fail: Permission denied
```
</details>

---

### Lab 5.2: Apply Seccomp Profile

**Scenario**: Apply a seccomp profile to restrict syscalls.

**Task**:
1. Create a seccomp profile at `/var/lib/kubelet/seccomp/profiles/audit.json`
2. Create a pod using RuntimeDefault seccomp
3. Create another pod using the custom profile

<details>
<summary>Solution</summary>

```bash
# Create seccomp profile directory
mkdir -p /var/lib/kubelet/seccomp/profiles

# Create audit profile
cat > /var/lib/kubelet/seccomp/profiles/audit.json <<EOF
{
  "defaultAction": "SCMP_ACT_LOG"
}
EOF

# Pod with RuntimeDefault
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-default
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
EOF

# Pod with custom profile
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-custom
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: app
    image: nginx
EOF
```

**Verify**:
```bash
kubectl get pod seccomp-default -o jsonpath='{.spec.securityContext.seccompProfile}'
kubectl get pod seccomp-custom -o jsonpath='{.spec.securityContext.seccompProfile}'
```
</details>

---

## Lab Set 6: Image Security and Scanning (Target: 15 minutes for all)

### Lab 6.1: Scan Images with Trivy

**Scenario**: Scan container images for vulnerabilities.

**Task**:
1. Scan `nginx:1.25` image
2. Find CRITICAL vulnerabilities
3. Scan `alpine:3.19` and compare

<details>
<summary>Solution</summary>

```bash
# Scan nginx
trivy image nginx:1.25

# Filter for CRITICAL only
trivy image --severity CRITICAL nginx:1.25

# Scan alpine
trivy image --severity CRITICAL,HIGH alpine:3.19

# Get count of vulnerabilities
trivy image nginx:1.25 2>/dev/null | grep -c CRITICAL
trivy image alpine:3.19 2>/dev/null | grep -c CRITICAL
```
</details>

---

### Lab 6.2: Find and Fix Vulnerable Pods

**Scenario**: Pods are running vulnerable images. Find and fix them.

```bash
# Setup
kubectl create namespace vuln-test
kubectl run vuln-nginx --image=nginx:1.14 -n vuln-test
kubectl run safe-nginx --image=nginx:1.25-alpine -n vuln-test
```

**Task**:
1. Scan images used by pods in `vuln-test` namespace
2. Identify which pod has more CRITICAL vulnerabilities
3. Delete the vulnerable pod

<details>
<summary>Solution</summary>

```bash
# Get images
kubectl get pods -n vuln-test -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[*].image}{"\n"}{end}'

# Scan each image
trivy image --severity CRITICAL nginx:1.14
trivy image --severity CRITICAL nginx:1.25-alpine

# Delete the more vulnerable pod
kubectl delete pod vuln-nginx -n vuln-test
```
</details>

---

## Lab Set 7: Audit Logging (Target: 15 minutes)

### Lab 7.1: Configure Audit Policy

**Scenario**: Configure audit logging to capture security-relevant events.

**Task**: Create an audit policy that:
1. Logs all requests to secrets at RequestResponse level
2. Logs pod exec/attach at RequestResponse level
3. Logs RBAC changes at RequestResponse level
4. Logs all other requests at Metadata level

<details>
<summary>Solution</summary>

```bash
# Create audit policy
mkdir -p /etc/kubernetes/audit

cat > /etc/kubernetes/audit/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log secrets access
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log pod exec/attach
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]
  
  # Log RBAC changes
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  
  # Log everything else at Metadata
  - level: Metadata
    omitStages:
      - RequestReceived
EOF

# Create log directory
mkdir -p /var/log/kubernetes/audit

# Add to kube-apiserver.yaml:
# - --audit-policy-file=/etc/kubernetes/audit/audit-policy.yaml
# - --audit-log-path=/var/log/kubernetes/audit/audit.log
# - --audit-log-maxage=30
# - --audit-log-maxbackup=10
# - --audit-log-maxsize=100
```
</details>

---

### Lab 7.2: Analyze Audit Logs

**Scenario**: Analyze audit logs to find suspicious activity.

**Task**: Using the audit logs, find:
1. All secret access events
2. All failed requests
3. All delete operations on pods

<details>
<summary>Solution</summary>

```bash
# All secret access
cat /var/log/kubernetes/audit/audit.log | jq 'select(.objectRef.resource=="secrets")'

# Failed requests (4xx/5xx)
cat /var/log/kubernetes/audit/audit.log | jq 'select(.responseStatus.code >= 400)'

# Delete operations on pods
cat /var/log/kubernetes/audit/audit.log | jq 'select(.verb=="delete" and .objectRef.resource=="pods")'

# Find requests by specific user
cat /var/log/kubernetes/audit/audit.log | jq 'select(.user.username=="system:anonymous")'

# Count requests per user
cat /var/log/kubernetes/audit/audit.log | jq -r '.user.username' | sort | uniq -c | sort -rn | head -10
```
</details>

---

## Lab Set 8: Falco (Target: 20 minutes)

### Lab 8.1: Install and Configure Falco

**Scenario**: Install Falco and configure it to detect threats.

**Task** (on a Kubernetes node):
1. Install Falco
2. Verify it's running
3. View default rules

<details>
<summary>Solution</summary>

```bash
# Install Falco
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" | sudo tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt-get update
sudo apt-get install -y falco

# Start Falco
sudo systemctl start falco
sudo systemctl enable falco

# Check status
sudo systemctl status falco

# View logs
journalctl -fu falco

# View rules
cat /etc/falco/falco_rules.yaml | head -100
```
</details>

---

### Lab 8.2: Create Custom Falco Rule

**Scenario**: Create a Falco rule to detect specific activities.

**Task**: Create a custom rule that:
1. Detects when `curl` or `wget` is run in a container
2. Outputs: timestamp, user, container_id, image, command
3. Priority: WARNING

<details>
<summary>Solution</summary>

```bash
# Create custom rule
cat > /etc/falco/rules.d/custom-rules.yaml <<EOF
- rule: Network download tools in container
  desc: Detect curl/wget usage in containers
  condition: >
    spawned_process and container and
    proc.name in (curl, wget)
  output: >
    Network tool executed 
    (time=%evt.time user=%user.name container_id=%container.id 
    image=%container.image.repository command=%proc.cmdline)
  priority: WARNING
  tags: [network, container]
EOF

# Restart Falco
systemctl restart falco

# Test: Run curl in a container
kubectl run test-curl --image=curlimages/curl -- sleep 3600
kubectl exec test-curl -- curl -s https://example.com

# Check Falco logs
journalctl -u falco --since "2 minutes ago" | grep "Network tool"
```
</details>

---

### Lab 8.3: Modify Falco Rule Output

**Scenario**: Modify an existing Falco rule to change its output format.

**Task**: Modify the "Terminal shell in container" rule to output only:
- Timestamp (without nanoseconds)
- Container ID
- Container image
- User name

<details>
<summary>Solution</summary>

```bash
# Find the original rule
grep -A 10 "Terminal shell in container" /etc/falco/falco_rules.yaml

# Create override in local rules
cat >> /etc/falco/falco_rules.local.yaml <<EOF

- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    %evt.time,%container.id,%container.image.repository,%user.name
  priority: WARNING
  tags: [container, shell]
EOF

# Restart Falco
systemctl restart falco

# Test
kubectl run shell-test --image=nginx -- sleep 3600
kubectl exec -it shell-test -- /bin/sh -c "exit"

# Check output format
cat /var/log/syslog | grep falco | tail -5
```
</details>

---

## Lab Set 9: RuntimeClass and Sandboxing (Target: 10 minutes)

### Lab 9.1: Create RuntimeClass

**Scenario**: Configure RuntimeClass for sandboxed workloads.

**Task**:
1. Create RuntimeClass for gVisor (handler: runsc)
2. Create a pod using this RuntimeClass

<details>
<summary>Solution</summary>

```bash
# Create RuntimeClass
kubectl apply -f - <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF

# Create pod with RuntimeClass
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-pod
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: nginx
EOF

# Note: This will only work if gVisor is installed on nodes
# For exam practice, understand the concept and YAML
```
</details>

---

## Lab Set 10: Certificate Management (Target: 15 minutes)

### Lab 10.1: Create User Certificate

**Scenario**: Create a certificate for a new user.

**Task**:
1. Generate key and CSR for user `john`
2. Create CertificateSigningRequest
3. Approve the CSR
4. Create kubeconfig for the user

<details>
<summary>Solution</summary>

```bash
# Generate key
openssl genrsa -out john.key 2048

# Generate CSR
openssl req -new -key john.key -out john.csr -subj "/CN=john/O=developers"

# Create CSR in Kubernetes
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-csr
spec:
  request: $(cat john.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
  expirationSeconds: 86400
EOF

# Approve CSR
kubectl certificate approve john-csr

# Get certificate
kubectl get csr john-csr -o jsonpath='{.status.certificate}' | base64 -d > john.crt

# Create kubeconfig entry
kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key

kubectl config set-context john-context \
  --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --user=john

# Test (will fail without RBAC)
kubectl --context=john-context get pods
```
</details>

---

## Cleanup Script

Run this between lab sets to clean up:

```bash
#!/bin/bash
# cleanup.sh

# Delete test namespaces
kubectl delete namespace production secure-apps dev app-ns secure-ns hardened fix-me secrets-lab vuln-test rbac-test np-test intern-ns --ignore-not-found

# Delete test pods in default namespace
kubectl delete pod --all -n default --ignore-not-found

# Delete test clusterrolebindings
kubectl delete clusterrolebinding suspicious-binding --ignore-not-found

# Reset
echo "Cleanup complete!"
```

---

## Progress Tracking

| Lab | Target Time | Your Time | Completed Without Docs | Date |
|-----|-------------|-----------|------------------------|------|
| 1.1 Default Deny | 3 min | | ☐ | |
| 1.2 Allow Traffic | 7 min | | ☐ | |
| 1.3 Block Metadata | 3 min | | ☐ | |
| 2.1 Developer Role | 5 min | | ☐ | |
| 2.2 ServiceAccount | 7 min | | ☐ | |
| 2.3 Fix Permissions | 5 min | | ☐ | |
| 3.1 PSS Enforcement | 5 min | | ☐ | |
| 3.2 Immutable Pod | 7 min | | ☐ | |
| 3.3 Fix Insecure Pod | 5 min | | ☐ | |
| 4.1 Secrets Volume | 5 min | | ☐ | |
| 4.2 Encryption | 10 min | | ☐ | |
| 5.1 AppArmor | 10 min | | ☐ | |
| 5.2 Seccomp | 7 min | | ☐ | |
| 6.1 Trivy Scan | 5 min | | ☐ | |
| 6.2 Fix Vulnerable | 7 min | | ☐ | |
| 7.1 Audit Policy | 10 min | | ☐ | |
| 7.2 Analyze Logs | 5 min | | ☐ | |
| 8.1 Install Falco | 10 min | | ☐ | |
| 8.2 Custom Rule | 7 min | | ☐ | |
| 8.3 Modify Rule | 5 min | | ☐ | |
| 9.1 RuntimeClass | 5 min | | ☐ | |
| 10.1 User Cert | 10 min | | ☐ | |

**Goal**: Complete all labs within target time without looking at solutions.