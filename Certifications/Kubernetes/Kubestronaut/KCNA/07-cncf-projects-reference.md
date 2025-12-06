# CNCF Projects Quick Reference for KCNA

## MUST KNOW: This appears frequently on the exam!

---

## Project Maturity Levels

| Level | Description | Requirements |
|-------|-------------|--------------|
| **Sandbox** | Early stage, experimental | Aligned with CNCF, 2 sponsors |
| **Incubating** | Growing, production use | Growing adoption, healthy project |
| **Graduated** | Mature, widely adopted | Proven adoption, stable governance |

---

## GRADUATED PROJECTS (Know ALL of These!)

### Container Orchestration & Runtime
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **Kubernetes** | Orchestration | Container orchestration platform | The core platform for KCNA |
| **containerd** | Runtime | Container runtime | Default runtime in K8s 1.24+ |
| **CRI-O** | Runtime | Kubernetes-specific runtime | Alternative to containerd |

### Networking
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **Envoy** | Proxy | L7 proxy and communication bus | Data plane for service meshes |
| **CoreDNS** | DNS | Kubernetes DNS server | Default DNS in Kubernetes |
| **Cilium** | CNI/Security | eBPF-based networking | Advanced networking + security |

### Service Mesh
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **Istio** | Service Mesh | Service mesh platform | Uses Envoy, full-featured |
| **Linkerd** | Service Mesh | Lightweight service mesh | Simple, Rust-based proxy |

### Observability
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **Prometheus** | Monitoring | Metrics and alerting | Pull-based, PromQL |
| **Jaeger** | Tracing | Distributed tracing | OpenTracing compatible |
| **Fluentd** | Logging | Log collector | Ruby-based, plugin ecosystem |
| **OpenTelemetry** | Observability | Unified telemetry | Merges OpenTracing + OpenCensus |

### Application Definition & Delivery
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **Helm** | Package Manager | Kubernetes package manager | Charts, values, templates |
| **Argo** | GitOps/CD | GitOps + Workflows | Argo CD for GitOps |
| **Flux** | GitOps | GitOps toolkit | Alternative to Argo CD |
| **Harbor** | Registry | Container registry | Vulnerability scanning |

### Storage
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **Rook** | Storage | Storage orchestration | Manages Ceph, etc. on K8s |
| **etcd** | Key-Value Store | Distributed KV store | Kubernetes backend |

### Security & Policy
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **Open Policy Agent (OPA)** | Policy | Policy engine | Rego language |
| **Falco** | Security | Runtime security | Threat detection |
| **cert-manager** | Certificates | Certificate management | Automates TLS certificates |
| **Notary** | Security | Content trust | Image signing |
| **TUF** | Security | Update framework | Secure software updates |

### Other Graduated
| Project | Category | Description | Key Points |
|---------|----------|-------------|------------|
| **SPIFFE/SPIRE** | Identity | Workload identity | Service identity |
| **Vitess** | Database | MySQL sharding | YouTube origin |
| **TiKV** | Database | Distributed KV store | Used by TiDB |
| **CloudEvents** | Messaging | Event data format | Standard for events |
| **KEDA** | Autoscaling | Event-driven autoscaling | Scale to zero |

---

## KEY INCUBATING PROJECTS

| Project | Category | Description |
|---------|----------|-------------|
| **Knative** | Serverless | Kubernetes serverless platform |
| **Backstage** | Developer Portal | Developer platform |
| **Dapr** | Runtime | Distributed application runtime |
| **Longhorn** | Storage | Cloud-native storage |
| **Kyverno** | Policy | Kubernetes-native policy |
| **Crossplane** | Infrastructure | Infrastructure as code via K8s |
| **OpenKruise** | Workloads | Advanced workload management |
| **KubeEdge** | Edge | Edge computing |

---

## Quick Category Reference

### Monitoring & Metrics
- **Prometheus** (Graduated) - Metrics collection and alerting
- **Grafana** (Not CNCF) - Visualization (often paired with Prometheus)
- **OpenTelemetry** (Graduated) - Unified observability

### Logging
- **Fluentd** (Graduated) - Full-featured log collector
- **Fluent Bit** (Part of Fluentd) - Lightweight log processor

### Tracing
- **Jaeger** (Graduated) - Distributed tracing
- **Zipkin** (Not CNCF) - Alternative tracing system

### Service Mesh
- **Istio** (Graduated) - Feature-rich, uses Envoy
- **Linkerd** (Graduated) - Simple, lightweight

### GitOps/CD
- **Argo CD** (Part of Argo, Graduated) - GitOps CD
- **Flux CD** (Graduated) - GitOps toolkit

### Policy & Security
- **OPA** (Graduated) - General policy engine
- **Kyverno** (Incubating) - Kubernetes-native policy
- **Falco** (Graduated) - Runtime security

### Networking
- **Cilium** (Graduated) - eBPF networking
- **Calico** (Not CNCF) - Popular CNI
- **Flannel** (Not CNCF) - Simple overlay

### Container Runtimes
- **containerd** (Graduated) - Industry standard
- **CRI-O** (Graduated) - Kubernetes-specific

---

## CNCF Landscape Categories

The [CNCF Landscape](https://landscape.cncf.io) organizes projects into:

1. **App Definition and Development**
   - Database, Streaming & Messaging
   - Application Definition & Image Build
   - CI/CD

2. **Orchestration & Management**
   - Scheduling & Orchestration
   - Coordination & Service Discovery
   - Service Mesh
   - API Gateway

3. **Runtime**
   - Cloud Native Storage
   - Container Runtime
   - Cloud Native Network

4. **Provisioning**
   - Automation & Configuration
   - Container Registry
   - Security & Compliance

5. **Observability and Analysis**
   - Monitoring
   - Logging
   - Tracing
   - Chaos Engineering

---

## Exam Question Patterns

### "Which CNCF project is used for...?"

| Question | Answer |
|----------|--------|
| Metrics collection? | Prometheus |
| Distributed tracing? | Jaeger |
| Log aggregation? | Fluentd |
| Service mesh? | Istio or Linkerd |
| GitOps? | Argo CD or Flux |
| Kubernetes package management? | Helm |
| Policy engine? | OPA or Kyverno |
| Runtime security? | Falco |
| Container registry? | Harbor |
| Certificate management? | cert-manager |
| Storage orchestration? | Rook |
| Event-driven autoscaling? | KEDA |
| Serverless? | Knative |

### "What maturity level is...?"

| Project | Level |
|---------|-------|
| Kubernetes | Graduated |
| Prometheus | Graduated |
| Helm | Graduated |
| Jaeger | Graduated |
| Istio | Graduated |
| Knative | Incubating |
| Backstage | Incubating |
| Kyverno | Incubating |

### "What is the purpose of...?"

| Project | Purpose |
|---------|---------|
| Envoy | High-performance L7 proxy (service mesh data plane) |
| CoreDNS | DNS server for Kubernetes |
| etcd | Distributed key-value store for cluster data |
| containerd | Container runtime |
| Cilium | eBPF-based networking, security, observability |
| SPIFFE | Secure workload identity |

---

## Memory Tricks

### "The Graduated 5" for Observability:
- **P**rometheus (metrics)
- **J**aeger (traces)
- **F**luentd (logs)
- **O**penTelemetry (unified)
- **G**rafana* (*not CNCF, but commonly used)

### Service Mesh = "IL" (I-L)
- **I**stio (feature-rich)
- **L**inkerd (lightweight)

### GitOps = "AF" (A-F)
- **A**rgo CD
- **F**lux CD

### Runtimes = "CC" (C-C)
- **C**ontainerd
- **C**RI-O

---

## Common Confusions to Avoid

| Confusion | Clarification |
|-----------|---------------|
| Prometheus vs Grafana | Prometheus = metrics collection; Grafana = visualization |
| Fluentd vs Fluent Bit | Fluentd = full-featured; Fluent Bit = lightweight |
| Istio vs Envoy | Istio = service mesh; Envoy = proxy used by Istio |
| Argo vs Argo CD | Argo = umbrella project; Argo CD = GitOps component |
| OPA vs Kyverno | OPA = general policy; Kyverno = K8s-native policy |
| Helm vs Kustomize | Helm = templates; Kustomize = overlays (no templates) |

---

## Not CNCF But Important to Know

| Tool | Category | Notes |
|------|----------|-------|
| Docker | Container Platform | Not CNCF (company) |
| Grafana | Visualization | Not CNCF (company) |
| Calico | CNI | Not CNCF (Tigera) |
| Flannel | CNI | Not CNCF (CoreOS) |
| Nginx | Ingress/Proxy | Not CNCF (F5) |
| Traefik | Ingress/Proxy | Not CNCF (Traefik Labs) |
| Terraform | IaC | Not CNCF (HashiCorp) |
| Vault | Secrets | Not CNCF (HashiCorp) |
| Consul | Service Discovery | Not CNCF (HashiCorp) |

---

## CNCF Official Links

- **CNCF Landscape**: https://landscape.cncf.io
- **CNCF Projects**: https://www.cncf.io/projects/
- **KCNA Curriculum**: https://github.com/cncf/curriculum

**Study the landscape before the exam - know where projects fit!**