# CKAD Exam: Complete Resource Guide for 100% Pass Rate

## The Reality of 100% Pass Probability

To maximize your chances, you need:
- **Knowledge** (the 5 study guides I created) ✓
- **Hands-on Practice** (50+ hours minimum)
- **Speed & Efficiency** (kubectl muscle memory)
- **Exam Simulation** (killer.sh + mock exams)
- **Exam Strategy** (time management)

---

## 1. OFFICIAL RESOURCES (Must Use)

### Allowed During Exam
These are the ONLY external resources allowed during the exam:

| Resource | URL | Purpose |
|----------|-----|---------|
| Kubernetes Docs | https://kubernetes.io/docs/ | Primary reference |
| Kubernetes Blog | https://kubernetes.io/blog/ | Updates |
| Helm Docs | https://helm.sh/docs/ | Helm commands |

**Bookmark these pages before the exam:**
- https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- https://kubernetes.io/docs/tasks/configure-pod-container/security-context/

### Official Curriculum
- https://github.com/cncf/curriculum (always check latest version)

---

## 2. HANDS-ON PRACTICE PLATFORMS (Critical)

### Free Platforms

| Platform | URL | Notes |
|----------|-----|-------|
| **Killercoda** | https://killercoda.com/ckad | Free CKAD scenarios |
| **Play with Kubernetes** | https://labs.play-with-k8s.com/ | Free 4-hour sessions |
| **Minikube** | https://minikube.sigs.k8s.io/ | Local cluster |
| **kind** | https://kind.sigs.k8s.io/ | Kubernetes in Docker |
| **K3s** | https://k3s.io/ | Lightweight K8s |

### Paid Platforms (Highly Recommended)

| Platform | URL | Notes |
|----------|-----|-------|
| **KodeKloud** | https://kodekloud.com/courses/certified-kubernetes-application-developer-ckad/ | Best course + labs |
| **killer.sh** | https://killer.sh/ckad | **Included with exam registration** (2 sessions) |
| **A Cloud Guru** | https://acloudguru.com/ | Good labs |

---

## 3. PRACTICE EXERCISES (Do All)

### GitHub Repositories

```bash
# Clone these repos and do EVERY exercise

# 1. dgkanatsios - Most popular CKAD exercises
git clone https://github.com/dgkanatsios/CKAD-exercises.git

# 2. bmuschko - Crash course exercises
git clone https://github.com/bmuschko/ckad-crash-course.git

# 3. lucassha - Additional practice
git clone https://github.com/lucassha/CKAD-resources.git
```

### Exercise Checklist

Complete each exercise **at least 3 times**:

- [ ] dgkanatsios: Core Concepts
- [ ] dgkanatsios: Multi-container Pods
- [ ] dgkanatsios: Pod Design
- [ ] dgkanatsios: Configuration
- [ ] dgkanatsios: Observability
- [ ] dgkanatsios: Services & Networking
- [ ] dgkanatsios: State Persistence
- [ ] dgkanatsios: Helm
- [ ] dgkanatsios: CRD
- [ ] bmuschko: All exercises
- [ ] KodeKloud: All labs (if enrolled)
- [ ] killer.sh: Both sessions (save for final week)

---

## 4. KUBECTL SPEED OPTIMIZATION (Essential)

### Bash Aliases (Set Up Immediately)

Add to `~/.bashrc` or exam terminal:

```bash
# Essential aliases
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias kds='kubectl describe svc'
alias kdd='kubectl describe deployment'
alias kl='kubectl logs'
alias kx='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'

# Context and namespace
alias kgc='kubectl config get-contexts'
alias kuc='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'

# Shortcuts for dry-run
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'

# Auto-completion
source <(kubectl completion bash)
complete -F __start_kubectl k
```

### Speed Commands to Memorize

```bash
# Generate YAML quickly (NEVER write from scratch)
k run nginx --image=nginx $do > pod.yaml
k create deploy nginx --image=nginx --replicas=3 $do > deploy.yaml
k create svc clusterip nginx --tcp=80:80 $do > svc.yaml
k create cm myconfig --from-literal=key=value $do > cm.yaml
k create secret generic mysecret --from-literal=pass=123 $do > secret.yaml
k create job myjob --image=busybox $do -- echo hello > job.yaml
k create cj mycron --image=busybox --schedule="*/5 * * * *" $do -- date > cj.yaml
k create ingress myingress --rule="host/path=svc:port" $do > ingress.yaml

# Fast resource operations
k expose deploy nginx --port=80 --target-port=8080
k scale deploy nginx --replicas=5
k set image deploy/nginx nginx=nginx:1.25
k rollout status deploy/nginx
k rollout undo deploy/nginx

# Quick debugging
k logs pod-name --previous
k exec -it pod-name -- /bin/sh
k describe pod pod-name | grep -A 10 Events

# Delete quickly
k delete pod nginx $now
```

### Vim Configuration

Add to `~/.vimrc`:

```vim
set tabstop=2
set shiftwidth=2
set expandtab
set number
set autoindent
syntax on
```

**Essential Vim commands:**
- `yy` - copy line
- `p` - paste
- `dd` - delete line
- `u` - undo
- `:set paste` - paste without auto-indent
- `:%s/old/new/g` - find and replace
- `/search` - search, `n` for next

---

## 5. EXAM STRATEGY (Critical for 100%)

### Time Management

| Task Weight | Time to Spend | Strategy |
|-------------|---------------|----------|
| 1-3% | 2-3 min | Do quickly or skip |
| 4-6% | 4-6 min | Standard effort |
| 7-10% | 7-10 min | Worth the time |
| 10%+ | 10-15 min | Prioritize these |

### Exam Flow

1. **First pass (60 min)**: Do all easy/medium questions
2. **Second pass (40 min)**: Tackle harder questions
3. **Final pass (20 min)**: Review flagged questions

### Critical Rules

```
✓ READ the question completely before starting
✓ Check the NAMESPACE in every question
✓ Verify the CONTEXT if multiple clusters
✓ Use imperative commands when possible
✓ Flag and skip if stuck > 5 minutes
✓ Validate your work: kubectl get, describe, logs
✓ Save time: copy YAML from docs, don't type
```

### Pre-Question Checklist

```bash
# Before EVERY question:
kubectl config use-context <given-context>
kubectl config set-context --current --namespace=<given-namespace>
```

---

## 6. MOCK EXAM SCHEDULE (2-Week Plan)

### Week 1: Foundation

| Day | Activity | Hours |
|-----|----------|-------|
| 1 | Review Domain 1 guide + dgkanatsios exercises | 3 |
| 2 | Review Domain 2 guide + dgkanatsios exercises | 3 |
| 3 | Review Domain 3 guide + dgkanatsios exercises | 2 |
| 4 | Review Domain 4 guide + dgkanatsios exercises | 4 |
| 5 | Review Domain 5 guide + dgkanatsios exercises | 3 |
| 6 | KodeKloud mock exam 1 (or free alternative) | 2 |
| 7 | Review weak areas, redo failed exercises | 3 |

### Week 2: Simulation

| Day | Activity | Hours |
|-----|----------|-------|
| 8 | Full practice exam (timed, 2 hours) | 2 |
| 9 | Review mistakes, practice weak areas | 3 |
| 10 | killer.sh Session 1 | 2 |
| 11 | Review killer.sh solutions thoroughly | 3 |
| 12 | killer.sh Session 2 | 2 |
| 13 | Light review, rest | 1 |
| 14 | **EXAM DAY** | - |

---

## 7. TOPIC CHECKLIST (Master Every Item)

### Domain 1: Application Design and Build (20%)
- [ ] Build container images with Dockerfile
- [ ] Multi-stage builds
- [ ] Create Deployment with replicas
- [ ] Create StatefulSet
- [ ] Create DaemonSet
- [ ] Create Job with completions/parallelism
- [ ] Create CronJob with schedule
- [ ] Init containers
- [ ] Sidecar containers
- [ ] emptyDir volumes
- [ ] PersistentVolumeClaim

### Domain 2: Application Deployment (20%)
- [ ] Rolling update with maxSurge/maxUnavailable
- [ ] Rollback deployment
- [ ] Pause/resume deployment
- [ ] Blue/green deployment (using labels)
- [ ] Canary deployment (using replicas)
- [ ] helm install/upgrade/rollback
- [ ] helm values override (--set, -f)
- [ ] kustomize base + overlays
- [ ] kubectl apply -k

### Domain 3: Application Observability and Maintenance (15%)
- [ ] Liveness probe (httpGet, exec, tcpSocket)
- [ ] Readiness probe
- [ ] Startup probe
- [ ] kubectl logs (--previous, --tail, -f, -c)
- [ ] kubectl top pods/nodes
- [ ] kubectl describe (read Events)
- [ ] kubectl exec into container
- [ ] Debug CrashLoopBackOff

### Domain 4: Application Environment, Configuration and Security (25%)
- [ ] Create ConfigMap (literal, file)
- [ ] Mount ConfigMap as env var
- [ ] Mount ConfigMap as volume
- [ ] Create Secret (generic, docker-registry, tls)
- [ ] Mount Secret as env var
- [ ] Mount Secret as volume
- [ ] Create ServiceAccount
- [ ] Assign ServiceAccount to Pod
- [ ] Create Role with verbs/resources
- [ ] Create RoleBinding
- [ ] kubectl auth can-i
- [ ] SecurityContext (runAsUser, runAsNonRoot)
- [ ] SecurityContext (readOnlyRootFilesystem, capabilities)
- [ ] Resource requests/limits
- [ ] ResourceQuota
- [ ] LimitRange

### Domain 5: Services and Networking (20%)
- [ ] Create ClusterIP Service
- [ ] Create NodePort Service
- [ ] Create LoadBalancer Service
- [ ] Debug Service (check endpoints)
- [ ] Create Ingress with path routing
- [ ] Create Ingress with host routing
- [ ] Ingress with TLS
- [ ] NetworkPolicy default deny
- [ ] NetworkPolicy allow specific pods
- [ ] NetworkPolicy with namespaceSelector

---

## 8. COMMON MISTAKES TO AVOID

| Mistake | Solution |
|---------|----------|
| Wrong namespace | Always check and set namespace first |
| Wrong context | Verify context before each question |
| Typing YAML manually | Use `--dry-run=client -o yaml` |
| Spending too long | Flag and move on after 5-7 min |
| Not validating | Always verify with `kubectl get/describe` |
| Forgetting labels | Selector must match pod labels |
| Wrong port | port=service, targetPort=container |
| Case sensitivity | YAML keys are case-sensitive |

---

## 9. EXAM DAY CHECKLIST

### Environment
- [ ] Stable internet connection
- [ ] Backup internet (mobile hotspot)
- [ ] Quiet room, cleared desk
- [ ] Government ID ready
- [ ] Webcam and microphone working
- [ ] Single monitor (or primary selected)
- [ ] Browser bookmarks ready

### Technical
- [ ] PSI Secure Browser installed
- [ ] System check completed
- [ ] Know how to copy/paste in exam terminal

### Mental
- [ ] Good night's sleep
- [ ] Light meal before
- [ ] Bathroom break before starting
- [ ] Stay calm, you've practiced

---

## 10. RESOURCES SUMMARY

### Must Do (Non-Negotiable)
1. ✅ My 5 study guides (knowledge foundation)
2. ✅ dgkanatsios exercises (3x each)
3. ✅ killer.sh (both sessions, final week)
4. ✅ kubectl aliases memorized
5. ✅ Kubernetes docs bookmarked

### Highly Recommended
- KodeKloud course + labs
- bmuschko exercises
- Killercoda scenarios

### Video Courses
- KodeKloud: Mumshad Mannambeth (Best for beginners)
- Udemy: Certified Kubernetes Application Developer
- LinkedIn Learning: CKAD Cert Prep

---

## Final Words

**The formula for 100%:**

```
Knowledge (guides) + Practice (50+ hours) + Speed (aliases) + Simulation (killer.sh) = PASS
```

**You will pass if you:**
1. Complete ALL exercises in dgkanatsios repo (3 times)
2. Score 90%+ on killer.sh sessions
3. Can create any resource in < 2 minutes
4. Know kubectl shortcuts by heart

**Good luck! You've got this! 🎯**