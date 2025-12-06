# CKS Domain 6: Monitoring, Logging, and Runtime Security (20%)

## Overview

This domain focuses on detecting threats and malicious activities at runtime. It covers behavioral analytics using tools like Falco, Kubernetes audit logging, ensuring container immutability, and investigating security incidents.

---

## 6.1 Perform Behavioral Analytics to Detect Malicious Activities

### Understanding Syscall Monitoring

Syscalls (system calls) are the interface between user-space applications and the kernel. Monitoring syscalls helps detect:

- Unexpected process execution (shells, package managers)
- Unauthorized file access
- Network connections
- Privilege escalation attempts

### Falco

Falco is the de facto runtime security tool for Kubernetes. It uses eBPF or kernel module to monitor syscalls and detect threats based on rules.

#### Installation

```bash
# Install Falco on a node
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/falcosecurity.list

sudo apt-get update
sudo apt-get install -y falco

# Start Falco
sudo systemctl start falco
sudo systemctl enable falco

# Check status
sudo systemctl status falco
```

#### Falco Configuration Files

```
/etc/falco/
├── falco.yaml              # Main configuration
├── falco_rules.yaml        # Default rules
├── falco_rules.local.yaml  # Custom rules (override defaults)
├── rules.d/                # Additional rule files
└── rules.available/        # Optional rule files
```

#### Main Configuration (/etc/falco/falco.yaml)

```yaml
# Output configuration
json_output: true
json_include_output_property: true

# Log output
syslog_output:
  enabled: true

file_output:
  enabled: true
  filename: /var/log/falco/events.log

stdout_output:
  enabled: true

# Program output (for alerts)
program_output:
  enabled: true
  program: "jq '{text: .output}' | curl -d @- -X POST https://hooks.slack.com/services/XXX"

# HTTP output
http_output:
  enabled: false
  url: http://some.url/

# Log level
log_level: info

# Rule files
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/rules.d
```

#### Falco Rule Structure

```yaml
- rule: <rule_name>
  desc: <description>
  condition: <filter_expression>
  output: <output_format>
  priority: <priority_level>
  tags: [<tag1>, <tag2>]
  enabled: <true|false>
```

#### Priority Levels

| Priority | Description |
|----------|-------------|
| EMERGENCY | System is unusable |
| ALERT | Action must be taken immediately |
| CRITICAL | Critical conditions |
| ERROR | Error conditions |
| WARNING | Warning conditions |
| NOTICE | Normal but significant |
| INFORMATIONAL | Informational messages |
| DEBUG | Debug-level messages |

#### Important Default Rules

```yaml
# Shell spawned in container
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: >
    A shell was spawned in a container with an attached terminal 
    (user=%user.name user_loginuid=%user.loginuid %container.info
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline
    terminal=%proc.tty container_id=%container.id image=%container.image.repository)
  priority: NOTICE
  tags: [container, shell, mitre_execution]

# Package management in container
- rule: Launch Package Management Process in Container
  desc: Package management process ran inside container
  condition: >
    spawned_process
    and container
    and user.name != "_apt"
    and package_mgmt_procs
    and not package_mgmt_ancestor_procs
    and not user_expected_package_management_in_container_conditions
  output: >
    Package management process launched in container 
    (user=%user.name user_loginuid=%user.loginuid command=%proc.cmdline
    container_id=%container.id container_name=%container.name
    image=%container.image.repository:%container.image.tag)
  priority: ERROR
  tags: [process, mitre_persistence]
```

#### Creating Custom Rules

```yaml
# /etc/falco/falco_rules.local.yaml

# Detect writes to /etc
- rule: Write to /etc in container
  desc: Detect any write to /etc directory in containers
  condition: >
    container and 
    open_write and 
    fd.directory = /etc
  output: >
    File opened for writing in /etc (user=%user.name user_loginuid=%user.loginuid 
    command=%proc.cmdline file=%fd.name container_id=%container.id 
    image=%container.image.repository)
  priority: WARNING
  tags: [filesystem, mitre_persistence]

# Detect sensitive file access
- rule: Read sensitive files
  desc: Detect read of sensitive files
  condition: >
    container and
    open_read and
    fd.name in (/etc/shadow, /etc/passwd, /etc/kubernetes)
  output: >
    Sensitive file read (user=%user.name file=%fd.name 
    container_id=%container.id image=%container.image.repository)
  priority: WARNING
  tags: [filesystem, mitre_credential_access]

# Detect network tool usage
- rule: Network tool in container
  desc: Detect network tools being used in container
  condition: >
    spawned_process and container and
    proc.name in (nc, ncat, nmap, netcat, curl, wget)
  output: >
    Network tool executed (user=%user.name command=%proc.cmdline 
    container_id=%container.id image=%container.image.repository)
  priority: NOTICE
  tags: [network, mitre_discovery]
```

#### Modifying Existing Rules

To modify a default rule, copy it to `falco_rules.local.yaml` and change it:

```yaml
# /etc/falco/falco_rules.local.yaml

# Override the default Terminal shell rule to change output
- rule: Terminal shell in container
  desc: A shell was used in a container
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    %evt.time,%user.name,%container.id,%container.image.repository
  priority: WARNING
  tags: [container, shell]
```

#### Useful Falco Commands

```bash
# Check Falco status
systemctl status falco

# View Falco logs
journalctl -fu falco

# View syslog output
cat /var/log/syslog | grep falco

# View file output
tail -f /var/log/falco/events.log

# Test rules with specific events
falco --list

# Validate rules
falco --validate /etc/falco/falco_rules.yaml

# Run Falco in foreground with verbose output
falco -o "stdout_output.enabled=true" -o "log_level=debug"

# List available fields
falco --list=syscall

# Restart Falco after rule changes
systemctl restart falco
```

#### Falco Output Fields

| Field | Description |
|-------|-------------|
| `%evt.time` | Event timestamp |
| `%user.name` | Username |
| `%user.uid` | User ID |
| `%container.id` | Container ID |
| `%container.name` | Container name |
| `%container.image.repository` | Image repository |
| `%proc.name` | Process name |
| `%proc.cmdline` | Full command line |
| `%proc.pname` | Parent process name |
| `%fd.name` | File descriptor name |

---

## 6.2 Use Kubernetes Audit Logs to Monitor Access

### Audit Logging Overview

Kubernetes audit logs record requests to the API server, providing a security-relevant chronological record of activities.

### Audit Stages

| Stage | Description |
|-------|-------------|
| RequestReceived | Event generated when request is received |
| ResponseStarted | Response headers sent, body not yet sent |
| ResponseComplete | Response body completed |
| Panic | Panic occurred during request handling |

### Audit Levels

| Level | Description |
|-------|-------------|
| None | Don't log events |
| Metadata | Log request metadata only |
| Request | Log metadata and request body |
| RequestResponse | Log metadata, request body, and response body |

### Creating an Audit Policy

```yaml
# /etc/kubernetes/audit/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Don't log read-only requests to certain resources
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: "" # core
        resources: ["endpoints", "services", "services/status"]
  
  # Don't log authenticated requests to certain non-resource URLs
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
      - "/api*"
      - "/version"
  
  # Log secrets access at Metadata level
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log pod changes at Request level
  - level: Request
    resources:
      - group: ""
        resources: ["pods"]
    verbs: ["create", "update", "patch", "delete"]
  
  # Log configmap and secret changes at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["configmaps", "secrets"]
    verbs: ["create", "update", "patch", "delete"]
  
  # Log everything else at Metadata level
  - level: Metadata
    omitStages:
      - RequestReceived
```

### Configuring the API Server

Add to `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    # Audit configuration
    - --audit-policy-file=/etc/kubernetes/audit/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    # Other flags...
    volumeMounts:
    - mountPath: /etc/kubernetes/audit
      name: audit-config
      readOnly: true
    - mountPath: /var/log/kubernetes/audit
      name: audit-log
  volumes:
  - name: audit-config
    hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
  - name: audit-log
    hostPath:
      path: /var/log/kubernetes/audit
      type: DirectoryOrCreate
```

### Audit Log Format

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "auditID": "unique-id",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/secrets",
  "verb": "create",
  "user": {
    "username": "admin",
    "uid": "admin-uid",
    "groups": ["system:masters", "system:authenticated"]
  },
  "sourceIPs": ["192.168.1.100"],
  "userAgent": "kubectl/v1.28.0",
  "objectRef": {
    "resource": "secrets",
    "namespace": "default",
    "name": "my-secret",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "metadata": {},
    "code": 201
  },
  "requestReceivedTimestamp": "2024-01-15T10:30:00.000000Z",
  "stageTimestamp": "2024-01-15T10:30:00.100000Z"
}
```

### Analyzing Audit Logs

```bash
# View recent audit logs
tail -f /var/log/kubernetes/audit/audit.log | jq

# Find all secret accesses
cat /var/log/kubernetes/audit/audit.log | jq 'select(.objectRef.resource=="secrets")'

# Find failed requests
cat /var/log/kubernetes/audit/audit.log | jq 'select(.responseStatus.code >= 400)'

# Find requests by specific user
cat /var/log/kubernetes/audit/audit.log | jq 'select(.user.username=="suspicious-user")'

# Find delete operations
cat /var/log/kubernetes/audit/audit.log | jq 'select(.verb=="delete")'

# Count requests by user
cat /var/log/kubernetes/audit/audit.log | jq -r '.user.username' | sort | uniq -c | sort -rn
```

### Audit Policy for Security Monitoring

```yaml
# /etc/kubernetes/audit/security-audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests to secrets
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log all privileged operations
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]
  
  # Log RBAC changes
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  
  # Log service account token requests
  - level: Metadata
    resources:
      - group: ""
        resources: ["serviceaccounts/token"]
  
  # Log authentication failures
  - level: Metadata
    nonResourceURLs:
      - "/api*"
    omitStages:
      - RequestReceived
  
  # Default: log metadata for all requests
  - level: Metadata
    omitStages:
      - RequestReceived
```

---

## 6.3 Ensure Immutability of Containers at Runtime

### Why Immutability?

- Prevents attackers from modifying container filesystem
- Ensures containers match their image
- Makes attacks more detectable
- Forces configuration to be declarative

### Read-Only Root Filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: immutable-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    # Mount writable volumes for required paths
    - name: tmp
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}
```

### Prevent Privilege Escalation

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-escalation-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      runAsNonRoot: true
      runAsUser: 1000
```

### Drop All Capabilities

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: minimal-caps-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      capabilities:
        drop:
        - ALL
```

### Complete Immutable Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fully-immutable-pod
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
    image: nginx:1.25-alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
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
    emptyDir:
      sizeLimit: "64Mi"
  - name: cache
    emptyDir:
      sizeLimit: "64Mi"
  - name: run
    emptyDir:
      sizeLimit: "1Mi"
```

### Enforce Immutability with Policy

Using Kyverno:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-ro-rootfs
spec:
  validationFailureAction: Enforce
  rules:
  - name: validate-readOnlyRootFilesystem
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Root filesystem must be read-only"
      pattern:
        spec:
          containers:
          - securityContext:
              readOnlyRootFilesystem: true
```

---

## 6.4 Investigate and Identify Threats

### Investigation Process

1. **Detection** - Alert from Falco, audit logs, or monitoring
2. **Collection** - Gather relevant logs and evidence
3. **Analysis** - Examine timeline and scope
4. **Containment** - Isolate affected resources
5. **Remediation** - Fix vulnerabilities and clean up
6. **Documentation** - Record findings and lessons learned

### Using crictl for Container Investigation

```bash
# List containers
crictl ps -a

# Get container details
crictl inspect <container-id>

# View container logs
crictl logs <container-id>

# Execute command in container
crictl exec -it <container-id> /bin/sh

# Get container stats
crictl stats <container-id>

# List images
crictl images

# Inspect image
crictl inspecti <image-id>
```

### Investigating Pods

```bash
# Get pod details
kubectl describe pod <pod-name>

# Get pod logs
kubectl logs <pod-name> --all-containers

# Get previous container logs
kubectl logs <pod-name> --previous

# Execute command in container
kubectl exec -it <pod-name> -- /bin/sh

# Get pod events
kubectl get events --field-selector involvedObject.name=<pod-name>

# Check pod security context
kubectl get pod <pod-name> -o jsonpath='{.spec.securityContext}'

# Check container security context
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].securityContext}'
```

### Investigating with /proc Filesystem

```bash
# Inside a container or on the node

# Get process environment variables
cat /proc/<pid>/environ | tr '\0' '\n'

# Get process command line
cat /proc/<pid>/cmdline | tr '\0' ' '

# Get open file descriptors
ls -la /proc/<pid>/fd

# Get process status
cat /proc/<pid>/status

# Get memory maps
cat /proc/<pid>/maps

# Network connections
cat /proc/net/tcp
cat /proc/net/tcp6
```

### Common Attack Indicators

| Indicator | Investigation Steps |
|-----------|---------------------|
| Unexpected shell process | Check Falco logs, audit logs |
| Package manager execution | Review container image, check modifications |
| Network tool usage | Analyze network policies, connections |
| Sensitive file access | Check audit logs for file access |
| New processes in container | Compare with container image |
| Outbound connections | Review network policies |

### Investigation Commands Summary

```bash
# Find suspicious processes
kubectl exec <pod> -- ps aux
kubectl exec <pod> -- cat /proc/1/cmdline

# Check for modifications
kubectl exec <pod> -- find / -mmin -60 -type f 2>/dev/null

# Check network connections
kubectl exec <pod> -- netstat -tulpn
kubectl exec <pod> -- ss -tulpn

# Check for installed packages
kubectl exec <pod> -- dpkg -l  # Debian/Ubuntu
kubectl exec <pod> -- rpm -qa  # RHEL/CentOS
kubectl exec <pod> -- apk list # Alpine

# Review Falco alerts
grep "container" /var/log/syslog | grep falco

# Review audit logs
cat /var/log/kubernetes/audit/audit.log | jq 'select(.user.username=="<suspect>")'
```

---

## Hands-On Lab Exercises

### Lab 1: Falco Rule Configuration

```bash
# SSH to a Kubernetes node with Falco installed

# Create custom rule file
cat > /etc/falco/rules.d/custom-rules.yaml <<EOF
- rule: Detect curl or wget in container
  desc: Detect network download tools in container
  condition: >
    spawned_process and container and
    proc.name in (curl, wget)
  output: >
    Network tool used (user=%user.name command=%proc.cmdline 
    container_id=%container.id image=%container.image.repository 
    time=%evt.time)
  priority: WARNING
  tags: [network, container]

- rule: Write to sensitive directory
  desc: Detect writes to /etc or /root
  condition: >
    container and
    open_write and
    (fd.directory = /etc or fd.directory = /root)
  output: >
    Write to sensitive dir (user=%user.name file=%fd.name 
    container_id=%container.id time=%evt.time)
  priority: ERROR
  tags: [filesystem, container]
EOF

# Restart Falco
systemctl restart falco

# Test the rules
# In another terminal, create a test pod
kubectl run test-curl --image=curlimages/curl -- sleep infinity

# Execute curl in the container
kubectl exec test-curl -- curl -s https://example.com

# Check Falco logs
journalctl -u falco --since "5 minutes ago" | grep "Network tool"
```

### Lab 2: Audit Policy Implementation

```bash
# Create audit policy
mkdir -p /etc/kubernetes/audit

cat > /etc/kubernetes/audit/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all secret access
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log pod exec
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec"]
  
  # Default
  - level: Metadata
    omitStages:
      - RequestReceived
EOF

# Create log directory
mkdir -p /var/log/kubernetes/audit

# Edit kube-apiserver manifest to add audit flags
# (Add to /etc/kubernetes/manifests/kube-apiserver.yaml)

# Wait for API server to restart
kubectl get pods -n kube-system -w

# Generate some audit events
kubectl get secrets
kubectl create secret generic test-secret --from-literal=key=value
kubectl exec <some-pod> -- ls

# Check audit logs
cat /var/log/kubernetes/audit/audit.log | jq 'select(.objectRef.resource=="secrets")' | head -50
```

### Lab 3: Container Immutability Verification

```bash
# Create an immutable pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: immutable-test
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: main
    image: busybox
    command: ["sleep", "infinity"]
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
EOF

# Test write protection
kubectl exec immutable-test -- touch /test-file
# Should fail: Read-only file system

kubectl exec immutable-test -- touch /tmp/test-file
# Should succeed (emptyDir mounted)

# Test privilege escalation
kubectl exec immutable-test -- id
# Should show non-root user
```

### Lab 4: Security Investigation

```bash
# Simulate an attack scenario
kubectl run attacker-pod --image=alpine -- sleep infinity
kubectl exec attacker-pod -- apk add curl nmap

# Investigation
# 1. Check Falco alerts
cat /var/log/syslog | grep falco | grep -i "package\|network"

# 2. Check audit logs for the pod
kubectl get pod attacker-pod -o jsonpath='{.metadata.uid}'
# Use the UID to search audit logs

# 3. Investigate the container
kubectl exec attacker-pod -- ps aux
kubectl exec attacker-pod -- cat /etc/apk/world  # Check installed packages
kubectl exec attacker-pod -- netstat -an

# 4. Check network activity
kubectl exec attacker-pod -- cat /proc/net/tcp

# 5. Cleanup
kubectl delete pod attacker-pod
```

---

## Practice Questions

1. **Create a Falco rule** that alerts when `/etc/passwd` is read in any container.

2. **Configure audit logging** to capture all requests to create or delete pods at the `RequestResponse` level.

3. **What command** shows the Falco rules currently loaded?

4. **Write a pod specification** that ensures container immutability with read-only root filesystem.

5. **How do you find** all audit events related to secret access by a specific user?

---

## Quick Reference

### Falco Commands

```bash
# Status
systemctl status falco

# Logs
journalctl -fu falco
cat /var/log/syslog | grep falco

# Validate rules
falco --validate <rules-file>

# List fields
falco --list

# Restart after changes
systemctl restart falco
```

### Falco Output Fields

| Field | Description |
|-------|-------------|
| `%evt.time` | Timestamp |
| `%user.name` | User |
| `%container.id` | Container ID |
| `%container.name` | Container name |
| `%container.image.repository` | Image |
| `%proc.name` | Process name |
| `%proc.cmdline` | Command line |
| `%fd.name` | File descriptor name |

### Audit Log Analysis

```bash
# All secret access
jq 'select(.objectRef.resource=="secrets")' audit.log

# Failed requests
jq 'select(.responseStatus.code >= 400)' audit.log

# Specific user
jq 'select(.user.username=="X")' audit.log

# Delete operations
jq 'select(.verb=="delete")' audit.log
```

### Immutability Security Context

```yaml
securityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  capabilities:
    drop:
    - ALL
```

---

## Official Documentation References

*These URLs are accessible during the CKS exam:*

- [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
- [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Configure a Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

*External resources (for study, not accessible during exam):*

- [Falco Documentation](https://falco.org/docs/)
- [Falco Rules](https://falco.org/docs/rules/)
- [Sysdig](https://docs.sysdig.com/)

---

## Exam Tips for This Domain

1. **Know Falco rule syntax** - You may need to modify rules to change output format
2. **Practice with audit logs** - Be comfortable with jq for parsing JSON logs
3. **Understand immutability** - Know which security context fields prevent modifications
4. **Investigation skills** - Be able to quickly identify suspicious activity
5. **Time management** - Don't spend too long on complex investigations

### Common Exam Scenarios

- Modify Falco rule to capture specific output fields
- Create audit policy to log specific resources
- Configure pod with read-only filesystem
- Find evidence of security incidents in logs
- Investigate suspicious container behavior