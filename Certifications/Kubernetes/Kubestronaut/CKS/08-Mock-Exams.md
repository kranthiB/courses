# CKS Mock Exams

## How to Use These Mock Exams

1. **Set a timer for 2 hours**
2. **No pausing** - simulate real exam conditions
3. **Only use allowed resources** (kubernetes.io/docs, falco.org/docs, trivy docs)
4. **Score yourself** using the point values
5. **Target score**: 67% to pass, aim for 85%+ before real exam

---

# Mock Exam 1 (100 points)

## Question 1 (7 points)

**Context**: Cluster `prod-cluster`

A NetworkPolicy named `existing-policy` in namespace `payments` is not working as expected. Pods with label `app=api` should only accept traffic from pods with label `app=frontend` on port 8080.

**Task**:
1. Analyze the existing NetworkPolicy
2. Fix any issues
3. Verify the policy works correctly

```bash
# Setup for practice
kubectl create namespace payments
kubectl run api --image=nginx -n payments -l app=api
kubectl run frontend --image=nginx -n payments -l app=frontend
kubectl run backend --image=nginx -n payments -l app=backend
kubectl expose pod api --port=8080 -n payments

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: existing-policy
  namespace: payments
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
          role: frontend  # Bug: wrong label
    ports:
    - port: 80  # Bug: wrong port
EOF
```

<details>
<summary>Solution</summary>

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: existing-policy
  namespace: payments
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
          app: frontend  # Fixed label
    ports:
    - port: 8080  # Fixed port
EOF

# Verify
kubectl exec -n payments frontend -- curl -s --max-time 3 api:8080
kubectl exec -n payments backend -- curl -s --max-time 3 api:8080  # Should fail
```
</details>

---

## Question 2 (8 points)

**Context**: Cluster `prod-cluster`

**Task**: Create a new ServiceAccount `restricted-sa` in namespace `secure-apps` that:
1. Does NOT automatically mount API credentials
2. Has a Role that only allows `get` and `list` on ConfigMaps
3. Create a pod `test-pod` using this ServiceAccount

<details>
<summary>Solution</summary>

```bash
kubectl create namespace secure-apps

kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: restricted-sa
  namespace: secure-apps
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: secure-apps
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: restricted-sa-binding
  namespace: secure-apps
subjects:
- kind: ServiceAccount
  name: restricted-sa
  namespace: secure-apps
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: secure-apps
spec:
  serviceAccountName: restricted-sa
  automountServiceAccountToken: false
  containers:
  - name: main
    image: nginx
EOF
```
</details>

---

## Question 3 (6 points)

**Context**: Cluster `prod-cluster`

**Task**: The namespace `web-team` needs to enforce the `baseline` Pod Security Standard. Configure the namespace to:
1. **Enforce** baseline level
2. **Warn** on restricted level violations
3. **Audit** at restricted level

<details>
<summary>Solution</summary>

```bash
kubectl create namespace web-team

kubectl label namespace web-team \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest
```
</details>

---

## Question 4 (8 points)

**Context**: Cluster `prod-cluster`

**Task**: Create a pod `secure-pod` in namespace `hardened` with the following security requirements:
- Run as user ID 1000, group ID 1000
- Read-only root filesystem
- No privilege escalation allowed
- Drop ALL capabilities
- Use RuntimeDefault seccomp profile
- Mount an emptyDir at `/tmp` for temporary files

<details>
<summary>Solution</summary>

```bash
kubectl create namespace hardened

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: hardened
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: main
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

## Question 5 (7 points)

**Context**: Cluster `prod-cluster`

**Task**: Use Trivy to scan the following images and identify which one has the LEAST number of CRITICAL vulnerabilities:
- `nginx:1.19`
- `nginx:1.25-alpine`
- `httpd:2.4`

Write the image name with least CRITICAL vulnerabilities to `/opt/answers/least-vulnerable.txt`

<details>
<summary>Solution</summary>

```bash
mkdir -p /opt/answers

# Scan each image
trivy image --severity CRITICAL nginx:1.19 2>/dev/null | grep -c CRITICAL
trivy image --severity CRITICAL nginx:1.25-alpine 2>/dev/null | grep -c CRITICAL
trivy image --severity CRITICAL httpd:2.4 2>/dev/null | grep -c CRITICAL

# nginx:1.25-alpine typically has fewest
echo "nginx:1.25-alpine" > /opt/answers/least-vulnerable.txt
```
</details>

---

## Question 6 (10 points)

**Context**: Node `worker-1`

Falco is installed on node `worker-1`. A container is spawning shells which is suspicious behavior.

**Task**:
1. SSH to `worker-1`
2. Find the Falco logs showing shell activity
3. Identify the container ID and image that spawned the shell
4. Save the container ID to `/opt/answers/suspicious-container.txt`

<details>
<summary>Solution</summary>

```bash
# SSH to worker node
ssh worker-1

# Check Falco logs
journalctl -u falco --since "10 minutes ago" | grep -i shell

# Or check syslog
cat /var/log/syslog | grep falco | grep -i shell

# Extract container ID from output and save
# Format: container_id=<id>
echo "<container-id>" > /opt/answers/suspicious-container.txt
```
</details>

---

## Question 7 (8 points)

**Context**: Node `worker-1`

**Task**: Create a custom Falco rule in `/etc/falco/rules.d/custom.yaml` that:
1. Detects when `kubectl` is executed inside a container
2. Rule name: `kubectl in container`
3. Priority: ERROR
4. Output format: `Kubectl executed (user=%user.name container=%container.id image=%container.image.repository command=%proc.cmdline)`

<details>
<summary>Solution</summary>

```bash
ssh worker-1

cat > /etc/falco/rules.d/custom.yaml <<EOF
- rule: kubectl in container
  desc: Detect kubectl execution inside containers
  condition: >
    spawned_process and container and
    proc.name = kubectl
  output: >
    Kubectl executed (user=%user.name container=%container.id 
    image=%container.image.repository command=%proc.cmdline)
  priority: ERROR
  tags: [container, kubectl]
EOF

# Restart Falco
systemctl restart falco

# Verify rule is loaded
falco --validate /etc/falco/rules.d/custom.yaml
```
</details>

---

## Question 8 (7 points)

**Context**: Cluster `prod-cluster`

**Task**: Create an Ingress resource `secure-ingress` in namespace `web` that:
1. Routes traffic for host `secure.example.com` to service `web-svc` on port 80
2. Uses TLS with secret `web-tls`
3. Forces HTTPS redirect (nginx annotation)

```bash
# Setup
kubectl create namespace web
kubectl create deployment web --image=nginx -n web
kubectl expose deployment web --name=web-svc --port=80 -n web
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key -n web  # Assume certs exist
```

<details>
<summary>Solution</summary>

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: web
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: web-tls
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
EOF
```
</details>

---

## Question 9 (6 points)

**Context**: Cluster `prod-cluster`

**Task**: A user `developer` should only be able to create and delete pods in namespace `dev-ns`. They should NOT be able to access secrets.

Create the appropriate RBAC configuration.

<details>
<summary>Solution</summary>

```bash
kubectl create namespace dev-ns

kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: dev-ns
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "delete", "get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-manager
  namespace: dev-ns
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify
kubectl auth can-i create pods -n dev-ns --as developer       # yes
kubectl auth can-i get secrets -n dev-ns --as developer       # no
```
</details>

---

## Question 10 (8 points)

**Context**: Cluster `prod-cluster`

**Task**: Create an audit policy at `/etc/kubernetes/audit/policy.yaml` that:
1. Logs all requests to secrets at `RequestResponse` level
2. Logs all `delete` operations at `RequestResponse` level
3. Logs everything else at `Metadata` level
4. Omits `RequestReceived` stage

<details>
<summary>Solution</summary>

```bash
mkdir -p /etc/kubernetes/audit

cat > /etc/kubernetes/audit/policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log secrets at RequestResponse
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log all deletes at RequestResponse
  - level: RequestResponse
    verbs: ["delete"]
  
  # Log everything else at Metadata
  - level: Metadata
    omitStages:
      - RequestReceived
EOF
```
</details>

---

## Question 11 (7 points)

**Context**: Cluster `prod-cluster`

**Task**: Configure encryption at rest for secrets:
1. Create EncryptionConfiguration at `/etc/kubernetes/enc/enc.yaml`
2. Use `aescbc` provider with a key named `key1`
3. Include identity provider as fallback

<details>
<summary>Solution</summary>

```bash
mkdir -p /etc/kubernetes/enc

# Generate key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

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

chmod 600 /etc/kubernetes/enc/enc.yaml

# Note: Would need to add --encryption-provider-config flag to API server
```
</details>

---

## Question 12 (6 points)

**Context**: Cluster `prod-cluster`

**Task**: Analyze the audit logs at `/var/log/kubernetes/audit.log` and find:
1. All requests that accessed secrets in `kube-system` namespace
2. Write the usernames who accessed these secrets to `/opt/answers/secret-users.txt`

<details>
<summary>Solution</summary>

```bash
cat /var/log/kubernetes/audit.log | \
  jq -r 'select(.objectRef.resource=="secrets" and .objectRef.namespace=="kube-system") | .user.username' | \
  sort | uniq > /opt/answers/secret-users.txt
```
</details>

---

## Question 13 (6 points)

**Context**: Cluster `prod-cluster`

**Task**: A pod is running in namespace `monitoring` with excessive privileges. Fix the security issues:

```bash
# Current pod (insecure)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: monitor-pod
  namespace: monitoring
spec:
  containers:
  - name: monitor
    image: nginx
    securityContext:
      privileged: true
      runAsUser: 0
EOF
```

<details>
<summary>Solution</summary>

```bash
kubectl delete pod monitor-pod -n monitoring

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: monitor-pod
  namespace: monitoring
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: monitor
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

## Question 14 (6 points)

**Context**: Node `worker-1`

**Task**: An AppArmor profile `k8s-restrict-write` exists on the node. Create a pod `apparmor-pod` in namespace `secure` that uses this profile for the container named `app`.

<details>
<summary>Solution</summary>

```bash
kubectl create namespace secure

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  namespace: secure
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-restrict-write
spec:
  containers:
  - name: app
    image: nginx
EOF
```
</details>

---

## Scoring

| Question | Points | Your Score |
|----------|--------|------------|
| Q1 - NetworkPolicy Fix | 7 | |
| Q2 - ServiceAccount | 8 | |
| Q3 - Pod Security Standards | 6 | |
| Q4 - Secure Pod | 8 | |
| Q5 - Trivy Scanning | 7 | |
| Q6 - Falco Investigation | 10 | |
| Q7 - Falco Custom Rule | 8 | |
| Q8 - Ingress TLS | 7 | |
| Q9 - RBAC | 6 | |
| Q10 - Audit Policy | 8 | |
| Q11 - Encryption at Rest | 7 | |
| Q12 - Audit Log Analysis | 6 | |
| Q13 - Fix Insecure Pod | 6 | |
| Q14 - AppArmor | 6 | |
| **Total** | **100** | |

**Passing Score**: 67 points

---

# Mock Exam 2 (100 points)

## Question 1 (8 points)

**Context**: Cluster `security-cluster`

**Task**: Create a NetworkPolicy `db-isolation` in namespace `database` that:
1. Selects pods with label `app=mysql`
2. Allows ingress only from pods with label `app=backend` on port 3306
3. Allows egress only to DNS (port 53 UDP)
4. Denies all other traffic

<details>
<summary>Solution</summary>

```bash
kubectl create namespace database

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-isolation
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - port: 3306
      protocol: TCP
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - port: 53
      protocol: UDP
EOF
```
</details>

---

## Question 2 (7 points)

**Context**: Cluster `security-cluster`

**Task**: Use kubesec to scan the pod definition at `/root/pod.yaml`. Fix any security issues with a score less than 0. Save the fixed manifest to `/root/pod-fixed.yaml`.

```yaml
# /root/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: web
    image: nginx
    securityContext:
      privileged: true
      capabilities:
        add:
        - SYS_ADMIN
```

<details>
<summary>Solution</summary>

```bash
# Scan original
kubesec scan /root/pod.yaml

# Create fixed version
cat > /root/pod-fixed.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: web
    image: nginx
    securityContext:
      privileged: false
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: "100m"
        memory: "128Mi"
      requests:
        cpu: "50m"
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
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
EOF

# Verify improved score
kubesec scan /root/pod-fixed.yaml
```
</details>

---

## Question 3 (8 points)

**Context**: Cluster `security-cluster`

**Task**: Find all ClusterRoleBindings that grant `cluster-admin` ClusterRole. Save the names of any suspicious bindings (not system bindings) to `/opt/answers/admin-bindings.txt`. Delete any that grant `cluster-admin` to non-system users.

<details>
<summary>Solution</summary>

```bash
# Find cluster-admin bindings
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name' > /tmp/all-admin-bindings.txt

# Filter out system bindings and save suspicious ones
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | select(.metadata.name | startswith("system:") | not) | .metadata.name' > /opt/answers/admin-bindings.txt

# Delete suspicious bindings (non-system)
for binding in $(cat /opt/answers/admin-bindings.txt); do
  kubectl delete clusterrolebinding $binding
done
```
</details>

---

## Question 4 (7 points)

**Context**: Cluster `security-cluster`

**Task**: Create a Secret `api-key` in namespace `app` containing `key=mysecretapikey123`. Create a Pod `api-pod` that:
1. Mounts the secret at `/etc/api/` as read-only
2. Sets file permission mode to 0400
3. Does NOT expose the secret as environment variables

<details>
<summary>Solution</summary>

```bash
kubectl create namespace app

kubectl create secret generic api-key -n app --from-literal=key=mysecretapikey123

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  namespace: app
spec:
  containers:
  - name: api
    image: nginx
    volumeMounts:
    - name: api-secret
      mountPath: /etc/api
      readOnly: true
  volumes:
  - name: api-secret
    secret:
      secretName: api-key
      defaultMode: 0400
EOF
```
</details>

---

## Question 5 (10 points)

**Context**: Node `node01`

**Task**: Modify the Falco rule "Terminal shell in container" to output the following format:
```
ALERT: %evt.time - Shell=%proc.name User=%user.name Container=%container.id Image=%container.image.repository
```

Save the modified rule in `/etc/falco/falco_rules.local.yaml` and restart Falco.

<details>
<summary>Solution</summary>

```bash
ssh node01

cat >> /etc/falco/falco_rules.local.yaml <<EOF

- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
  output: >
    ALERT: %evt.time - Shell=%proc.name User=%user.name Container=%container.id Image=%container.image.repository
  priority: WARNING
  tags: [container, shell]
EOF

systemctl restart falco

# Verify
journalctl -u falco --since "1 minute ago"
```
</details>

---

## Question 6 (6 points)

**Context**: Cluster `security-cluster`

**Task**: Create a RuntimeClass `secure-runtime` with handler `runsc`. Then create a pod `sandboxed-app` in namespace `sandbox` that uses this RuntimeClass.

<details>
<summary>Solution</summary>

```bash
kubectl apply -f - <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: secure-runtime
handler: runsc
EOF

kubectl create namespace sandbox

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-app
  namespace: sandbox
spec:
  runtimeClassName: secure-runtime
  containers:
  - name: app
    image: nginx
EOF
```
</details>

---

## Question 7 (8 points)

**Context**: Cluster `security-cluster`

**Task**: A CertificateSigningRequest `csr-user1` exists but is pending. 
1. Examine the CSR to find the username
2. Approve the CSR
3. Extract the certificate and save to `/opt/answers/user1.crt`

<details>
<summary>Solution</summary>

```bash
# Examine CSR
kubectl get csr csr-user1 -o yaml
kubectl get csr csr-user1 -o jsonpath='{.spec.request}' | base64 -d | openssl req -text -noout

# Approve
kubectl certificate approve csr-user1

# Extract certificate
kubectl get csr csr-user1 -o jsonpath='{.status.certificate}' | base64 -d > /opt/answers/user1.crt
```
</details>

---

## Question 8 (7 points)

**Context**: Cluster `security-cluster`

**Task**: Pods in namespace `public` should not be able to access the cloud metadata service at 169.254.169.254. Create a NetworkPolicy to block this.

<details>
<summary>Solution</summary>

```bash
kubectl create namespace public

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata
  namespace: public
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

## Question 9 (8 points)

**Context**: Cluster `security-cluster`

**Task**: Create an audit policy that:
1. Does NOT log requests to endpoints or services by `system:kube-proxy`
2. Logs `RequestResponse` for secrets
3. Logs `RequestResponse` for RBAC resources
4. Logs `Metadata` for everything else

Save to `/etc/kubernetes/audit/audit-policy.yaml`

<details>
<summary>Solution</summary>

```bash
mkdir -p /etc/kubernetes/audit

cat > /etc/kubernetes/audit/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Don't log kube-proxy watching endpoints/services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services"]
  
  # Log secrets at RequestResponse
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log RBAC at RequestResponse
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["*"]
  
  # Everything else at Metadata
  - level: Metadata
    omitStages:
      - RequestReceived
EOF
```
</details>

---

## Question 10 (6 points)

**Context**: Cluster `security-cluster`

**Task**: Create a pod `seccomp-pod` in namespace `restricted` that uses the `RuntimeDefault` seccomp profile at the pod level.

<details>
<summary>Solution</summary>

```bash
kubectl create namespace restricted

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
  namespace: restricted
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
EOF
```
</details>

---

## Question 11 (7 points)

**Context**: Cluster `security-cluster`

**Task**: Multiple pods are running in namespace `scan-me`. Use Trivy to identify which pod is using an image with CRITICAL vulnerabilities. Delete that pod.

```bash
# Setup
kubectl create namespace scan-me
kubectl run safe-app --image=alpine:3.19 -n scan-me
kubectl run vuln-app --image=nginx:1.14 -n scan-me
kubectl run another-safe --image=busybox:1.36 -n scan-me
```

<details>
<summary>Solution</summary>

```bash
# Get images
kubectl get pods -n scan-me -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[*].image}{"\n"}{end}'

# Scan each
trivy image --severity CRITICAL alpine:3.19
trivy image --severity CRITICAL nginx:1.14
trivy image --severity CRITICAL busybox:1.36

# Delete the vulnerable pod
kubectl delete pod vuln-app -n scan-me
```
</details>

---

## Question 12 (6 points)

**Context**: Cluster `security-cluster`

**Task**: The kubelet on `node01` is configured insecurely. SSH to the node and:
1. Disable anonymous authentication
2. Set authorization mode to Webhook
3. Restart kubelet

<details>
<summary>Solution</summary>

```bash
ssh node01

# Edit kubelet config
cat > /var/lib/kubelet/config.yaml <<EOF
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
# ... other existing settings
EOF

systemctl restart kubelet
systemctl status kubelet
```
</details>

---

## Question 13 (6 points)

**Context**: Cluster `security-cluster`

**Task**: Verify the SHA256 checksum of kubectl binary at `/usr/local/bin/kubectl`. Compare it with the official checksum for version v1.31.0. Write "VALID" or "INVALID" to `/opt/answers/kubectl-check.txt`

<details>
<summary>Solution</summary>

```bash
# Get installed binary hash
sha256sum /usr/local/bin/kubectl

# Download official checksum
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl.sha256"

# Compare and write result
INSTALLED=$(sha256sum /usr/local/bin/kubectl | awk '{print $1}')
OFFICIAL=$(cat kubectl.sha256)

if [ "$INSTALLED" == "$OFFICIAL" ]; then
  echo "VALID" > /opt/answers/kubectl-check.txt
else
  echo "INVALID" > /opt/answers/kubectl-check.txt
fi
```
</details>

---

## Question 14 (6 points)

**Context**: Node `node01`

**Task**: Use Falco to find which container modified `/etc/passwd`. Save the container ID to `/opt/answers/passwd-modifier.txt`

<details>
<summary>Solution</summary>

```bash
ssh node01

# Search Falco logs for /etc/passwd modifications
cat /var/log/syslog | grep falco | grep "/etc/passwd" | grep -i "write\|modify\|open"

# Or use journalctl
journalctl -u falco | grep "/etc/passwd"

# Extract container ID and save
echo "<container-id>" > /opt/answers/passwd-modifier.txt
```
</details>

---

## Scoring

| Question | Points | Your Score |
|----------|--------|------------|
| Q1 - NetworkPolicy DB | 8 | |
| Q2 - Kubesec Fix | 7 | |
| Q3 - Audit ClusterRoleBindings | 8 | |
| Q4 - Secret Mount | 7 | |
| Q5 - Falco Rule Modification | 10 | |
| Q6 - RuntimeClass | 6 | |
| Q7 - CSR Approval | 8 | |
| Q8 - Block Metadata | 7 | |
| Q9 - Audit Policy | 8 | |
| Q10 - Seccomp Pod | 6 | |
| Q11 - Trivy + Delete | 7 | |
| Q12 - Kubelet Security | 6 | |
| Q13 - Binary Verification | 6 | |
| Q14 - Falco Investigation | 6 | |
| **Total** | **100** | |

**Passing Score**: 67 points

---

## Mock Exam Tips

1. **Read each question completely** before starting
2. **Note the namespace** - many mistakes happen here
3. **Use `--dry-run=client -o yaml`** to generate templates
4. **Test your work** - verify pods are running, policies are working
5. **Manage your time** - don't spend more than 8-10 minutes on any question
6. **Flag and skip** difficult questions, come back later
7. **Use kubectl explain** for YAML structure help