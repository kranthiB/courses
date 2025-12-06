# CKS 14-Day Study Plan for 100% Pass Probability

## Overview

This plan assumes you have:
- Valid CKA certification (prerequisite)
- 2-3 hours daily study time
- Access to a practice cluster (Minikube, kind, or cloud)
- Killer.sh simulator (included with exam registration)

---

## Week 1: Foundation & Core Concepts

### Day 1: Setup & Cluster Security Basics
**Morning (1.5 hours)**
- [ ] Read: CKS-Domain1-Cluster-Setup.md (full)
- [ ] Setup practice environment (Minikube with NetworkPolicy support)
- [ ] Configure aliases and bash completion

**Evening (1.5 hours)**
- [ ] Lab: Network Policy exercises 1.1-1.3
- [ ] Speed Drill: Set 3 (NetworkPolicy) - 3 repetitions
- [ ] Practice: Default deny + allow specific traffic patterns

**Checkpoint**: Can you create a NetworkPolicy from memory?

---

### Day 2: RBAC Deep Dive
**Morning (1.5 hours)**
- [ ] Read: CKS-Domain2-Cluster-Hardening.md (full)
- [ ] Focus sections: RBAC, ServiceAccounts

**Evening (1.5 hours)**
- [ ] Lab: RBAC exercises 2.1-2.3
- [ ] Speed Drill: Set 2 (RBAC) - 3 repetitions
- [ ] Practice: Create Role, ClusterRole, bindings imperatively

**Checkpoint**: Can you explain the difference between Role and ClusterRole?

---

### Day 3: ServiceAccounts & API Security
**Morning (1.5 hours)**
- [ ] Review: ServiceAccount sections
- [ ] Read: API server security configuration
- [ ] Study: Token mounting, RBAC for SAs

**Evening (1.5 hours)**
- [ ] Lab: ServiceAccount exercises
- [ ] Speed Drill: Set 4 (Secrets) - 3 repetitions
- [ ] Practice: Create SA with minimal permissions

**Checkpoint**: Can you create a ServiceAccount that doesn't auto-mount tokens?

---

### Day 4: System Hardening
**Morning (1.5 hours)**
- [ ] Read: CKS-Domain3-System-Hardening.md (full)
- [ ] Focus: AppArmor, Seccomp concepts

**Evening (1.5 hours)**
- [ ] Lab: AppArmor exercise 5.1
- [ ] Lab: Seccomp exercise 5.2
- [ ] Speed Drill: Set 9 (AppArmor) - 3 repetitions

**Checkpoint**: Can you create and load an AppArmor profile?

---

### Day 5: Pod Security Deep Dive
**Morning (1.5 hours)**
- [ ] Read: CKS-Domain4-Minimize-Microservice-Vulnerabilities.md
- [ ] Focus: Pod Security Standards, Security Contexts

**Evening (1.5 hours)**
- [ ] Lab: Pod Security exercises 3.1-3.3
- [ ] Speed Drill: Set 5 (Security Context) - 5 repetitions
- [ ] Speed Drill: Set 6 (PSA Labels) - 3 repetitions

**Checkpoint**: Can you write a Restricted-compliant pod from memory?

---

### Day 6: Secrets & Encryption
**Morning (1.5 hours)**
- [ ] Review: Secrets management section
- [ ] Study: Encryption at rest configuration
- [ ] Read: etcd encryption setup

**Evening (1.5 hours)**
- [ ] Lab: Secrets exercises 4.1-4.2
- [ ] Practice: Configure encryption at rest
- [ ] Speed Drill: Set 4 - 3 repetitions

**Checkpoint**: Can you configure encryption at rest for secrets?

---

### Day 7: Week 1 Review & Practice
**Morning (2 hours)**
- [ ] Review all Week 1 materials
- [ ] Complete any unfinished labs
- [ ] Speed drills for all sets learned (1-6, 9)

**Evening (2 hours)**
- [ ] Complete Mock Exam 1 (Questions 1-7 only)
- [ ] Time yourself - target 50 minutes for 7 questions
- [ ] Review mistakes thoroughly

**Checkpoint**: Score at least 70% on partial mock exam

---

## Week 2: Advanced Topics & Exam Preparation

### Day 8: Supply Chain Security
**Morning (1.5 hours)**
- [ ] Read: CKS-Domain5-Supply-Chain-Security.md (full)
- [ ] Focus: Image scanning, Trivy usage

**Evening (1.5 hours)**
- [ ] Lab: Trivy scanning exercises 6.1-6.2
- [ ] Speed Drill: Set 7 (Trivy) - 3 repetitions
- [ ] Practice: Scan images, identify CVEs

**Checkpoint**: Can you use Trivy to find critical vulnerabilities?

---

### Day 9: Static Analysis & Image Security
**Morning (1.5 hours)**
- [ ] Study: Kubesec, Dockerfile best practices
- [ ] Review: Image registry restrictions
- [ ] Read: OPA/Kyverno basics

**Evening (1.5 hours)**
- [ ] Lab: Kubesec scanning
- [ ] Practice: Fix insecure Dockerfiles
- [ ] Practice: Create image restriction policies

**Checkpoint**: Can you analyze and fix a Dockerfile for security issues?

---

### Day 10: Runtime Security & Falco
**Morning (1.5 hours)**
- [ ] Read: CKS-Domain6-Monitoring-Logging-Runtime-Security.md (full)
- [ ] Focus: Falco rules, configuration

**Evening (1.5 hours)**
- [ ] Lab: Falco exercises 8.1-8.3
- [ ] Speed Drill: Set 8 (Falco) - 5 repetitions
- [ ] Practice: Modify Falco rules, change output format

**Checkpoint**: Can you write and modify Falco rules?

---

### Day 11: Audit Logging
**Morning (1.5 hours)**
- [ ] Review: Audit policy configuration
- [ ] Study: Audit log analysis with jq

**Evening (1.5 hours)**
- [ ] Lab: Audit exercises 7.1-7.2
- [ ] Speed Drill: Set 10 (Audit Logs) - 3 repetitions
- [ ] Practice: Write audit policies, analyze logs

**Checkpoint**: Can you create an audit policy and analyze logs?

---

### Day 12: Full Mock Exams
**Morning (2 hours)**
- [ ] Complete Mock Exam 1 (full, timed - 2 hours)
- [ ] Do NOT look at solutions during exam

**Afternoon (1 hour)**
- [ ] Review all answers
- [ ] Identify weak areas
- [ ] Re-do failed questions

**Evening (1 hour)**
- [ ] Speed drills for weak areas
- [ ] Review relevant study guide sections

**Checkpoint**: Score at least 75% on Mock Exam 1

---

### Day 13: Mock Exam 2 & Killer.sh
**Morning (2 hours)**
- [ ] Complete Mock Exam 2 (full, timed)
- [ ] Review answers

**Afternoon (2 hours)**
- [ ] **Killer.sh Simulator - Session 1**
- [ ] Treat it like real exam
- [ ] Note difficult topics

**Evening (1 hour)**
- [ ] Review killer.sh solutions
- [ ] Make notes on tricky questions

**Checkpoint**: Score at least 70% on killer.sh

---

### Day 14: Final Preparation
**Morning (2 hours)**
- [ ] Review all weak areas from mock exams
- [ ] Speed drills - all sets once
- [ ] Quick review of all study guides

**Afternoon (2 hours)**
- [ ] **Killer.sh Simulator - Session 2**
- [ ] Focus on speed and accuracy

**Evening (1 hour)**
- [ ] Light review only
- [ ] Prepare exam environment
- [ ] Rest well!

**Final Checkpoint**: Score at least 80% on killer.sh

---

## Exam Day Checklist

### Before Exam
- [ ] ID ready
- [ ] Room clean, clear desk
- [ ] Water bottle (clear)
- [ ] Stable internet connection
- [ ] Webcam and mic working
- [ ] PSI browser installed

### First 5 Minutes
- [ ] Set up aliases:
```bash
alias k=kubectl
export do='--dry-run=client -o yaml'
source <(kubectl completion bash)
complete -F __start_kubectl k
```
- [ ] Verify cluster access
- [ ] Read through all questions (note point values)

### During Exam
- [ ] Start with high-point questions you're confident about
- [ ] Flag difficult questions, move on
- [ ] Don't spend more than 8 minutes on any question
- [ ] Verify your work before moving on
- [ ] Use `k explain` for YAML structure help

### Time Management
| Time | Activity |
|------|----------|
| 0-5 min | Setup, read questions |
| 5-60 min | High-confidence questions |
| 60-100 min | Medium difficulty |
| 100-115 min | Review flagged questions |
| 115-120 min | Final verification |

---

## Score Tracking

| Assessment | Target | Your Score | Date |
|------------|--------|------------|------|
| Mock Exam 1 (partial) | 70% | | |
| Mock Exam 1 (full) | 75% | | |
| Mock Exam 2 | 75% | | |
| Killer.sh Session 1 | 70% | | |
| Killer.sh Session 2 | 80% | | |
| **Real Exam** | **67%** | | |

---

## Topic Confidence Tracker

Rate yourself 1-5 after each study session:

| Topic | Day 1 | Day 7 | Day 14 |
|-------|-------|-------|--------|
| Network Policies | | | |
| RBAC | | | |
| ServiceAccounts | | | |
| Pod Security Standards | | | |
| Security Context | | | |
| Secrets Management | | | |
| Encryption at Rest | | | |
| AppArmor | | | |
| Seccomp | | | |
| Trivy | | | |
| Falco | | | |
| Audit Logging | | | |
| CIS Benchmarks | | | |
| Ingress TLS | | | |

**Scale**: 1=No idea, 2=Basic, 3=Can do with docs, 4=Can do quickly, 5=Expert

---

## Resources Quick Reference

### Allowed During Exam
- https://kubernetes.io/docs/
- https://kubernetes.io/blog/
- https://github.com/kubernetes/
- https://falco.org/docs/
- https://aquasecurity.github.io/trivy/

### Practice Platforms
- Killer.sh (included with exam)
- KodeKloud CKS course
- Killercoda free scenarios

### Quick Links to Bookmark
1. [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
2. [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
3. [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
4. [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
5. [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
6. [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
7. [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
8. [Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)

---

## Emergency Cheat Sheet (Review Night Before)

### Network Policy Structure
```yaml
spec:
  podSelector:
    matchLabels:
      app: X
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: Y
    ports:
    - port: 80
```

### Security Context Essentials
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

### PSA Labels
```
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/warn: restricted
pod-security.kubernetes.io/audit: restricted
```

### Falco Rule Structure
```yaml
- rule: Name
  condition: spawned_process and container and proc.name = X
  output: "Alert %user.name %container.id"
  priority: WARNING
```

### Quick Commands
```bash
# RBAC check
k auth can-i <verb> <resource> --as <user>

# Decode secret
k get secret X -o jsonpath='{.data.key}' | base64 -d

# Trivy scan
trivy image --severity CRITICAL <image>

# Falco logs
journalctl -fu falco
```

---

**You've got this! Trust your preparation and stay calm during the exam.**