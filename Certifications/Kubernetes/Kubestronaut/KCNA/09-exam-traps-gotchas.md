# KCNA Common Exam Traps and Gotchas

## 🚨 CRITICAL: These Trick Questions Catch Many People!

---

## TRAP #1: Secrets Are NOT Encrypted by Default

**Wrong Answer Trap**: "Secrets are encrypted in Kubernetes"

**Correct Answer**: Secrets are **base64 ENCODED**, not encrypted!

**What you need to know**:
- Base64 is encoding, NOT encryption
- Anyone can decode base64
- Encryption at rest must be configured separately
- Secrets are slightly more secure than ConfigMaps (not shown in `kubectl get`), but still readable

**Example Exam Question**:
> How are Kubernetes Secrets stored by default?
- A) Encrypted with AES-256
- B) Hashed with SHA-256  
- C) Base64 encoded ✓
- D) Plain text

---

## TRAP #2: kube-scheduler Does NOT Run Pods

**Wrong Answer Trap**: "The scheduler runs Pods on nodes"

**Correct Answer**: kube-scheduler only **ASSIGNS** Pods to nodes. The **kubelet** runs them.

**The flow**:
1. kube-scheduler: Decides WHERE to run
2. kubelet: Actually RUNS the container

---

## TRAP #3: ReplicaSet vs Deployment

**Wrong Answer Trap**: "Use ReplicaSet to manage Pods"

**Correct Answer**: Use **Deployment** to manage Pods. Deployments manage ReplicaSets.

**When asked about best practice**:
- Pod → Deployment (not direct)
- ReplicaSet → Deployment (not direct)
- StatefulSets → For stateful apps

---

## TRAP #4: ClusterIP is DEFAULT Service Type

**Wrong Answer Trap**: "NodePort is the default"

**Correct Answer**: **ClusterIP** is the default Service type.

| Type | Default? | Access |
|------|----------|--------|
| ClusterIP | YES ✓ | Internal only |
| NodePort | No | External via node port |
| LoadBalancer | No | External via LB |

---

## TRAP #5: CoreDNS (Not kube-dns)

**Wrong Answer Trap**: "kube-dns is the DNS server"

**Correct Answer**: **CoreDNS** is the default DNS since Kubernetes 1.13

**kube-dns** was the old default, CoreDNS replaced it.

---

## TRAP #6: containerd (Not Docker) is Default Runtime

**Wrong Answer Trap**: "Docker is the default container runtime"

**Correct Answer**: **containerd** is the default since Kubernetes 1.24

- Docker support (dockershim) was REMOVED in v1.24
- containerd or CRI-O are now used
- Docker images still work (OCI standard)

---

## TRAP #7: NoSchedule vs NoExecute Taints

**Wrong Answer Trap**: Confusing what each taint effect does

**Correct Answers**:
| Effect | New Pods | Existing Pods |
|--------|----------|---------------|
| NoSchedule | Won't schedule | NOT affected |
| PreferNoSchedule | Tries to avoid | NOT affected |
| NoExecute | Won't schedule | **EVICTED** |

**Key**: Only **NoExecute** affects existing pods!

---

## TRAP #8: Liveness vs Readiness Probes

**Wrong Answer Trap**: Confusing which probe does what

**Correct Answers**:
| Probe | Purpose | On Failure |
|-------|---------|------------|
| Liveness | Is container alive? | RESTART container |
| Readiness | Ready for traffic? | REMOVE from Service |
| Startup | Has app started? | Disable other probes |

**Memory trick**:
- Liveness = Life (restart if dead)
- Readiness = Ready for traffic (no traffic if not ready)

---

## TRAP #9: Ingress vs Ingress Controller

**Wrong Answer Trap**: "Ingress routes traffic"

**Correct Answer**: **Ingress** is just a RULE. **Ingress Controller** does the routing.

- Ingress = Resource (rules)
- Ingress Controller = Pod that implements rules (nginx, traefik, etc.)
- You need BOTH!

---

## TRAP #10: Prometheus Pull Model

**Wrong Answer Trap**: "Applications push metrics to Prometheus"

**Correct Answer**: Prometheus uses a **PULL** model (scrapes targets)

- Prometheus SCRAPES metrics from targets
- Targets expose `/metrics` endpoint
- PushGateway is used for short-lived jobs (exception)

---

## TRAP #11: Horizontal vs Vertical Autoscaling

**Wrong Answer Trap**: Confusing HPA and VPA

**Correct Answers**:
| Autoscaler | What it scales |
|------------|----------------|
| HPA (Horizontal) | Number of Pod replicas |
| VPA (Vertical) | CPU/memory of Pods |
| Cluster Autoscaler | Number of nodes |

---

## TRAP #12: GitOps Source of Truth

**Wrong Answer Trap**: "Kubernetes cluster is the source of truth"

**Correct Answer**: In GitOps, **Git** is the source of truth

- Git repository defines desired state
- GitOps tool (Argo CD, Flux) syncs to cluster
- Cluster state should match Git

---

## TRAP #13: CNCF Maturity Levels

**Wrong Answer Trap**: Using wrong maturity terms

**Correct Levels** (in order):
1. **Sandbox** (early stage)
2. **Incubating** (growing)
3. **Graduated** (mature)

**NOT**: Alpha, Beta, Stable (that's Kubernetes features)

---

## TRAP #14: Istio vs Linkerd Differences

**Wrong Answer Trap**: Saying they're the same

**Key Differences**:
| Aspect | Istio | Linkerd |
|--------|-------|---------|
| Proxy | Envoy | Linkerd-proxy (Rust) |
| Complexity | More complex | Simpler |
| Features | More features | Core features |
| Resource usage | Higher | Lower |

---

## TRAP #15: Namespace-Scoped vs Cluster-Scoped

**Wrong Answer Trap**: Thinking all resources are namespace-scoped

**Cluster-scoped resources** (NOT in a namespace):
- Nodes
- PersistentVolumes
- ClusterRoles
- ClusterRoleBindings
- Namespaces themselves
- StorageClasses

**Namespace-scoped**:
- Pods, Deployments, Services
- ConfigMaps, Secrets
- Roles, RoleBindings
- PersistentVolumeClaims

---

## TRAP #16: Blue-Green vs Canary

**Wrong Answer Trap**: Confusing the two strategies

| Strategy | How it works |
|----------|--------------|
| Blue-Green | 100% traffic switch between environments |
| Canary | Gradual traffic shift (10%, 30%, 50%...) |

---

## TRAP #17: ConfigMap Changes Don't Restart Pods

**Wrong Answer Trap**: "Pods automatically reload ConfigMap changes"

**Correct Answer**: 
- ConfigMap changes do NOT restart Pods
- Mounted volumes may eventually update
- Environment variables require Pod restart
- Use tools like Reloader for automatic restarts

---

## TRAP #18: etcd is NOT Part of Worker Nodes

**Wrong Answer Trap**: Including etcd in worker node components

**Control Plane Components**:
- kube-apiserver
- etcd ✓
- kube-scheduler
- kube-controller-manager

**Worker Node Components**:
- kubelet
- kube-proxy
- Container runtime

---

## TRAP #19: Service Mesh Data Plane vs Control Plane

**Wrong Answer Trap**: Confusing what Envoy does

**Correct**:
- **Control Plane**: Configuration, policy (istiod)
- **Data Plane**: Traffic handling (Envoy sidecars)

Envoy is the **data plane** proxy!

---

## TRAP #20: OpenTelemetry Purpose

**Wrong Answer Trap**: "OpenTelemetry is a monitoring tool"

**Correct Answer**: OpenTelemetry is a **standard/framework** for:
- Generating telemetry data (metrics, logs, traces)
- NOT a backend/storage system
- Vendor-neutral
- Merged OpenTracing + OpenCensus

---

## Exam Day Tips

### Before the Exam
1. ✅ Get a good night's sleep
2. ✅ Review this document one more time
3. ✅ Have a clear workspace
4. ✅ Test your internet connection
5. ✅ Have your ID ready

### During the Exam
1. ✅ Read questions CAREFULLY
2. ✅ Look for keywords: "default", "best practice", "NOT"
3. ✅ Eliminate obviously wrong answers first
4. ✅ Flag uncertain questions and return later
5. ✅ Don't spend too long on any question
6. ✅ Trust your preparation

### Common Question Keywords

| Keyword | What to look for |
|---------|------------------|
| "default" | ClusterIP, RollingUpdate, containerd |
| "best practice" | Use Deployments, use namespaces, follow 12-factor |
| "NOT" | Read carefully - asking for wrong answer |
| "CNCF graduated" | Know the graduated project list |
| "source of truth" | Git (for GitOps) |

---

## Quick Recall Checklist

Before the exam, make sure you know:

- [ ] Control plane components (5)
- [ ] Worker node components (3)
- [ ] Service types (4)
- [ ] Pod Security Standards levels (3)
- [ ] CNCF maturity levels (3)
- [ ] Three pillars of observability
- [ ] Golden signals (4)
- [ ] Deployment strategies (4)
- [ ] HPA vs VPA vs Cluster Autoscaler
- [ ] Helm vs Kustomize differences
- [ ] Argo CD vs Flux CD
- [ ] Istio vs Linkerd
- [ ] Prometheus pull model
- [ ] GitOps principles

---

**Master these traps = Pass with flying colors! 🎯**