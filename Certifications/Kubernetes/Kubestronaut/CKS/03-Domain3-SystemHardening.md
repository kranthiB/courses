# CKS Domain 3: System Hardening (15%)

## Overview

This domain focuses on improving the security of the host operating system, minimizing the attack surface, restricting access through IAM, and appropriately using kernel hardening tools like AppArmor and seccomp.

---

## 3.1 Minimize Host OS Footprint (Reduce Attack Surface)

### Host Hardening Principles

1. **Use minimal base OS**: Alpine, CoreOS, Bottlerocket, Flatcar
2. **Remove unnecessary packages**
3. **Disable unnecessary services**
4. **Keep systems updated**
5. **Implement proper access controls**

### Identify and Remove Unnecessary Packages

```bash
# List installed packages (Debian/Ubuntu)
dpkg -l | wc -l
apt list --installed

# Remove unnecessary packages
apt remove --purge <package-name>
apt autoremove

# List installed packages (RHEL/CentOS)
rpm -qa | wc -l
yum list installed

# Remove packages
yum remove <package-name>
```

### Identify and Disable Unnecessary Services

```bash
# List all running services
systemctl list-units --type=service --state=running

# Disable unnecessary services
systemctl stop <service-name>
systemctl disable <service-name>

# Mask service to prevent starting
systemctl mask <service-name>

# Common services that may be unnecessary on Kubernetes nodes:
# - cups (printing)
# - avahi-daemon (zeroconf)
# - bluetooth
# - rpcbind
```

### Identify Open Ports

```bash
# List listening ports
ss -tulpn
netstat -tulpn

# Using nmap for comprehensive scan
nmap -sT -O localhost

# Check for unexpected listeners
ss -tulpn | grep -v ':22\|:6443\|:10250\|:10251\|:10252\|:2379\|:2380'
```

### File System Security

```bash
# Mount options for security
# /etc/fstab entries
/dev/sda1  /tmp     ext4  defaults,noexec,nosuid,nodev  0 2
/dev/sda2  /var     ext4  defaults,nosuid               0 2
/dev/sda3  /home    ext4  defaults,nodev                0 2

# Verify mount options
mount | grep -E '/tmp|/var|/home'

# Key mount options:
# noexec - Prevent execution of binaries
# nosuid - Ignore SUID/SGID bits
# nodev  - Ignore device files
```

### User and Access Management

```bash
# List users with shell access
grep -v '/nologin\|/false' /etc/passwd

# List users with UID 0 (root)
awk -F: '$3 == 0 {print $1}' /etc/passwd

# Lock unnecessary accounts
usermod -L <username>
passwd -l <username>

# Set account expiration
chage -E 0 <username>

# Remove unnecessary users
userdel -r <username>
```

### SSH Hardening

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
X11Forwarding no
MaxAuthTries 3
AllowUsers admin deploy
Protocol 2
ClientAliveInterval 300
ClientAliveCountMax 2

# Restart SSH
systemctl restart sshd
```

---

## 3.2 Minimize IAM Roles

### Principle of Least Privilege

For cloud-managed Kubernetes (EKS, GKE, AKS), minimize IAM roles assigned to nodes and service accounts.

### AWS EKS IAM Best Practices

```bash
# Create minimal node IAM policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeRegions",
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

### IAM Roles for Service Accounts (IRSA)

```yaml
# ServiceAccount with IAM role annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/s3-reader-role
```

### GKE Workload Identity

```bash
# Enable Workload Identity
gcloud container clusters update CLUSTER_NAME \
  --workload-pool=PROJECT_ID.svc.id.goog

# Create GCP service account
gcloud iam service-accounts create GSA_NAME

# Bind KSA to GSA
gcloud iam service-accounts add-iam-policy-binding GSA_NAME@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"
```

```yaml
# Annotate Kubernetes ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-ksa
  annotations:
    iam.gke.io/gcp-service-account: GSA_NAME@PROJECT_ID.iam.gserviceaccount.com
```

---

## 3.3 Minimize External Access to the Network

### Network-Level Security

```bash
# Configure iptables for node protection
# Allow established connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow localhost
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH (be careful!)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow Kubernetes components
iptables -A INPUT -p tcp --dport 6443 -s 10.0.0.0/8 -j ACCEPT   # API server
iptables -A INPUT -p tcp --dport 10250 -s 10.0.0.0/8 -j ACCEPT  # Kubelet
iptables -A INPUT -p tcp --dport 10251 -s 10.0.0.0/8 -j ACCEPT  # Scheduler
iptables -A INPUT -p tcp --dport 10252 -s 10.0.0.0/8 -j ACCEPT  # Controller

# Default deny
iptables -A INPUT -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### UFW Configuration

```bash
# Enable UFW
ufw enable

# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow specific ports
ufw allow from 10.0.0.0/8 to any port 22
ufw allow from 10.0.0.0/8 to any port 6443
ufw allow from 10.0.0.0/8 to any port 10250

# Check status
ufw status verbose
```

---

## 3.4 Appropriately Use Kernel Hardening Tools

### AppArmor

AppArmor (Application Armor) is a Linux Security Module that restricts programs' capabilities with per-program profiles.

#### Check AppArmor Status

```bash
# Check if AppArmor is enabled
aa-status
cat /sys/module/apparmor/parameters/enabled

# List loaded profiles
aa-status --profiles

# Check profile mode (enforce/complain)
aa-status --verbose
```

#### AppArmor Profile Modes

- **enforce**: Profile rules are enforced
- **complain**: Violations are logged but not blocked
- **unconfined**: No restrictions

#### Creating an AppArmor Profile

```bash
# /etc/apparmor.d/deny-write

#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>
  
  # Allow reading everything
  file,
  
  # Deny all file writes
  deny /** w,
}
```

#### Loading AppArmor Profiles

```bash
# Load profile in enforce mode
apparmor_parser -r /etc/apparmor.d/deny-write

# Load profile in complain mode
apparmor_parser -C /etc/apparmor.d/deny-write

# Remove profile
apparmor_parser -R /etc/apparmor.d/deny-write

# List profiles
aa-status
```

#### Using AppArmor with Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/secure-container: localhost/k8s-deny-write
spec:
  containers:
  - name: secure-container
    image: nginx
```

**Important**: The profile must be loaded on the node where the pod runs.

#### Example: Restrict Shell Execution

```bash
# /etc/apparmor.d/k8s-deny-exec

#include <tunables/global>

profile k8s-deny-exec flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny execution of shells
  deny /bin/sh mrwklx,
  deny /bin/bash mrwklx,
  deny /bin/dash mrwklx,
  deny /usr/bin/sh mrwklx,
  deny /usr/bin/bash mrwklx,
}
```

### Seccomp

Seccomp (Secure Computing Mode) restricts the system calls a process can make.

#### Check Seccomp Support

```bash
# Check if seccomp is enabled in kernel
grep SECCOMP /boot/config-$(uname -r)

# Check kubelet seccomp default
cat /var/lib/kubelet/config.yaml | grep -i seccomp
```

#### Seccomp Profile Locations

Default location for custom seccomp profiles:
```
/var/lib/kubelet/seccomp/profiles/
```

#### Creating a Seccomp Profile

```json
{
  "defaultAction": "SCMP_ACT_LOG",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "accept4",
        "arch_prctl",
        "bind",
        "brk",
        "clone",
        "close",
        "connect",
        "dup2",
        "epoll_create1",
        "epoll_ctl",
        "epoll_pwait",
        "execve",
        "exit_group",
        "fcntl",
        "fstat",
        "futex",
        "getdents64",
        "getpid",
        "getppid",
        "getsockname",
        "getsockopt",
        "ioctl",
        "listen",
        "lseek",
        "mmap",
        "mprotect",
        "munmap",
        "nanosleep",
        "newfstatat",
        "openat",
        "pipe2",
        "prctl",
        "pread64",
        "prlimit64",
        "read",
        "recvfrom",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sched_getaffinity",
        "sched_yield",
        "sendto",
        "set_robust_list",
        "set_tid_address",
        "setsockopt",
        "socket",
        "uname",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Save as `/var/lib/kubelet/seccomp/profiles/audit.json`

#### Using Seccomp with Pods (Kubernetes 1.19+)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: secure-container
    image: nginx
```

#### Seccomp Profile Types

| Type | Description |
|------|-------------|
| `RuntimeDefault` | Use container runtime's default profile |
| `Unconfined` | No seccomp restrictions (default) |
| `Localhost` | Use a profile from the node's filesystem |

#### Using RuntimeDefault Seccomp

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-default-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: secure-container
    image: nginx
```

#### Container-Level Seccomp

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-seccomp
spec:
  containers:
  - name: container1
    image: nginx
    securityContext:
      seccompProfile:
        type: RuntimeDefault
  - name: container2
    image: busybox
    command: ["sleep", "infinity"]
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/custom.json
```

---

## 3.5 Using SELinux (Alternative to AppArmor)

SELinux (Security-Enhanced Linux) is another Linux Security Module commonly used on RHEL-based systems.

### Check SELinux Status

```bash
# Check if SELinux is enabled
getenforce
sestatus

# Modes: Enforcing, Permissive, Disabled
```

### SELinux Pod Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selinux-pod
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"
      user: "system_u"
      role: "system_r"
      type: "container_t"
  containers:
  - name: secure-container
    image: nginx
```

---

## 3.6 Kernel Parameter Hardening

### Sysctl Security Settings

```bash
# /etc/sysctl.d/99-kubernetes-security.conf

# Disable IP forwarding (if not needed for routing)
# Note: Kubernetes requires this to be enabled
# net.ipv4.ip_forward = 0

# Prevent IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Enable SYN flood protection
net.ipv4.tcp_syncookies = 1

# Disable IPv6 if not needed
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Apply settings
sysctl -p /etc/sysctl.d/99-kubernetes-security.conf
```

### Protected Kubelet Sysctl

```yaml
# /var/lib/kubelet/config.yaml
protectKernelDefaults: true
allowedUnsafeSysctls:
- "net.core.somaxconn"
- "kernel.msg*"
```

### Pod Sysctl Settings

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-pod
spec:
  securityContext:
    sysctls:
    - name: net.core.somaxconn
      value: "1024"
    - name: kernel.shm_rmid_forced
      value: "1"
  containers:
  - name: main
    image: nginx
```

---

## Hands-On Lab Exercises

### Lab 1: AppArmor Profile Implementation

```bash
# Create AppArmor profile
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
  name: apparmor-test
  annotations:
    container.apparmor.security.beta.kubernetes.io/test-container: localhost/k8s-deny-write
spec:
  containers:
  - name: test-container
    image: nginx
    command: ["sh", "-c", "sleep infinity"]
EOF

# Test write restriction
kubectl exec apparmor-test -- touch /tmp/test-file
# Should fail: touch: cannot touch '/tmp/test-file': Permission denied
```

### Lab 2: Seccomp Profile Implementation

```bash
# Create seccomp directory if needed
mkdir -p /var/lib/kubelet/seccomp/profiles

# Create a seccomp profile that logs all syscalls
cat > /var/lib/kubelet/seccomp/profiles/audit.json <<EOF
{
  "defaultAction": "SCMP_ACT_LOG"
}
EOF

# Create pod with seccomp profile
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-test
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: test-container
    image: nginx
EOF

# Check audit logs for syscalls
# (logs will be in system audit log or dmesg)
dmesg | grep -i seccomp
```

### Lab 3: RuntimeDefault Seccomp

```bash
# Create pod with RuntimeDefault seccomp
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: runtime-default-seccomp
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: main
    image: nginx
EOF

# Verify pod is running
kubectl get pod runtime-default-seccomp

# Check the seccomp profile in use
kubectl get pod runtime-default-seccomp -o jsonpath='{.spec.securityContext.seccompProfile}'
```

### Lab 4: Host System Audit

```bash
# Audit open ports
ss -tulpn

# Check for unnecessary services
systemctl list-units --type=service --state=running

# Check user accounts
cat /etc/passwd | grep -v nologin | grep -v false

# Check SSH configuration
grep -E "PermitRootLogin|PasswordAuthentication" /etc/ssh/sshd_config

# Check file permissions on sensitive files
ls -la /etc/shadow
ls -la /etc/passwd
ls -la /etc/kubernetes/

# Check for SUID binaries
find / -perm -4000 -type f 2>/dev/null
```

---

## Practice Questions

1. **Create an AppArmor profile** that denies execution of `/bin/sh` and `/bin/bash`, then apply it to a pod.

2. **Configure a pod** to use the `RuntimeDefault` seccomp profile.

3. **What command** shows all loaded AppArmor profiles and their mode?

4. **Create a seccomp profile** that allows only `read`, `write`, `close`, and `exit_group` syscalls.

5. **How do you verify** that a container is running with a specific AppArmor profile?

---

## Quick Reference

### AppArmor Commands

```bash
# Check status
aa-status

# Load profile (enforce mode)
apparmor_parser -r /etc/apparmor.d/<profile>

# Load profile (complain mode)
apparmor_parser -C /etc/apparmor.d/<profile>

# Remove profile
apparmor_parser -R /etc/apparmor.d/<profile>

# Set profile to complain mode
aa-complain /etc/apparmor.d/<profile>

# Set profile to enforce mode
aa-enforce /etc/apparmor.d/<profile>
```

### Seccomp Actions

| Action | Description |
|--------|-------------|
| `SCMP_ACT_ALLOW` | Allow the syscall |
| `SCMP_ACT_ERRNO` | Return error code |
| `SCMP_ACT_KILL` | Kill the process |
| `SCMP_ACT_LOG` | Log the syscall |
| `SCMP_ACT_TRAP` | Throw SIGSYS signal |

### AppArmor Pod Annotation Format

```
container.apparmor.security.beta.kubernetes.io/<container-name>: <profile-ref>
```

Profile reference formats:
- `runtime/default` - Runtime default profile
- `localhost/<profile-name>` - Profile loaded on node
- `unconfined` - No AppArmor restrictions

### Seccomp Profile Locations

```
/var/lib/kubelet/seccomp/profiles/
```

---

## Official Documentation References

*These URLs are accessible during the CKS exam:*

- [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
- [Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [Pod Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Sysctls](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
- [Linux Kernel Security Constraints](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)