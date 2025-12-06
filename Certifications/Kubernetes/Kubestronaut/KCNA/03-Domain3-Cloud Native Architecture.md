# KCNA Exam Study Guide: Domain 3 - Cloud Native Architecture

## Domain Weight: 16% (~10 questions out of 60)

This domain covers cloud native principles, architectural patterns, the CNCF ecosystem, and key concepts like microservices, serverless, and autoscaling.

---

## 3.1 What is Cloud Native?

### CNCF Definition
> "Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach."

### Key Characteristics of Cloud Native
1. **Containerized**: Applications packaged in containers
2. **Dynamically orchestrated**: Managed by orchestration platforms (Kubernetes)
3. **Microservices-oriented**: Loosely coupled services
4. **Scalable**: Horizontal scaling on demand
5. **Resilient**: Self-healing and fault-tolerant
6. **Observable**: Comprehensive monitoring and tracing
7. **Automated**: CI/CD, infrastructure as code

### Cloud Native vs Traditional Applications

| Aspect | Traditional | Cloud Native |
|--------|-------------|--------------|
| **Architecture** | Monolithic | Microservices |
| **Deployment** | Physical/VMs | Containers |
| **Scaling** | Vertical (scale up) | Horizontal (scale out) |
| **Updates** | Infrequent, big releases | Frequent, small changes |
| **Infrastructure** | Mutable servers | Immutable infrastructure |
| **State** | Stateful servers | Stateless services |
| **Configuration** | Config files on disk | Environment variables, ConfigMaps |

---

## 3.2 Cloud Native Computing Foundation (CNCF)

### What is CNCF?
- **Purpose**: Foster the growth of cloud native technologies
- **Part of**: Linux Foundation
- **Founded**: 2015
- **Role**: Hosts and promotes cloud native projects

### CNCF Project Maturity Levels

```
┌─────────────────────────────────────────────────────────────┐
│                     GRADUATED                               │
│  (Production-ready, widely adopted, proven governance)      │
│  Examples: Kubernetes, Prometheus, Envoy, Helm, containerd  │
└─────────────────────────────────────────────────────────────┘
                          ▲
┌─────────────────────────────────────────────────────────────┐
│                     INCUBATING                              │
│  (Growing community, production use, maturing governance)   │
│  Examples: Argo, Backstage, Dapr, Knative, KEDA             │
└─────────────────────────────────────────────────────────────┘
                          ▲
┌─────────────────────────────────────────────────────────────┐
│                      SANDBOX                                │
│  (Early stage, experimental, innovation projects)           │
│  Examples: New and emerging projects                        │
└─────────────────────────────────────────────────────────────┘
```

### Key CNCF Graduated Projects

| Project | Category | Description |
|---------|----------|-------------|
| **Kubernetes** | Orchestration | Container orchestration platform |
| **Prometheus** | Monitoring | Metrics and alerting toolkit |
| **Envoy** | Proxy | High-performance L7 proxy |
| **CoreDNS** | DNS | Kubernetes DNS server |
| **containerd** | Runtime | Container runtime |
| **Helm** | Package Management | Kubernetes package manager |
| **etcd** | Key-Value Store | Distributed key-value store |
| **Fluentd** | Logging | Log collector and aggregator |
| **Jaeger** | Tracing | Distributed tracing system |
| **Argo** | GitOps/Workflows | GitOps and workflows for Kubernetes |
| **Istio** | Service Mesh | Service mesh platform |
| **Linkerd** | Service Mesh | Lightweight service mesh |
| **Open Policy Agent** | Policy | Policy engine for cloud native |
| **Cilium** | Networking | eBPF-based networking and security |
| **OpenTelemetry** | Observability | Observability framework |

### CNCF Landscape
- **URL**: https://landscape.cncf.io
- **Purpose**: Interactive map of cloud native technologies
- **Categories**: App Definition, Orchestration, Runtime, Provisioning, Observability, etc.
- **Important for KCNA**: Know major project categories and their purposes

---

## 3.3 Microservices Architecture

### What are Microservices?
- **Definition**: Architectural style where applications are composed of small, independent services
- **Communication**: Services communicate via APIs (REST, gRPC, messaging)
- **Deployment**: Each service can be deployed independently

### Monolithic vs Microservices

**Monolithic Architecture:**
```
┌─────────────────────────────────────────┐
│             MONOLITHIC APP              │
│  ┌───────────────────────────────────┐  │
│  │           User Interface          │  │
│  ├───────────────────────────────────┤  │
│  │         Business Logic            │  │
│  │  (All features in one codebase)   │  │
│  ├───────────────────────────────────┤  │
│  │          Data Access              │  │
│  └───────────────────────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │           Database                │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**Microservices Architecture:**
```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │  Order   │  │ Payment  │  │ Shipping │
│ Service  │  │ Service  │  │ Service  │  │ Service  │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │
     ▼             ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │  Order   │  │ Payment  │  │ Shipping │
│    DB    │  │    DB    │  │    DB    │  │    DB    │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### Microservices Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Single Responsibility** | Each service does one thing well |
| **Independently Deployable** | Deploy without affecting other services |
| **Decentralized Data** | Each service owns its data |
| **Technology Diversity** | Different languages/frameworks per service |
| **Fault Isolation** | Failure in one service doesn't cascade |
| **Scalable** | Scale individual services based on demand |

### Benefits of Microservices
- ✅ Independent deployment and scaling
- ✅ Technology flexibility
- ✅ Improved fault isolation
- ✅ Smaller, focused teams
- ✅ Easier to understand individual services

### Challenges of Microservices
- ❌ Distributed system complexity
- ❌ Network latency and failures
- ❌ Data consistency across services
- ❌ Testing and debugging complexity
- ❌ Operational overhead

### The Twelve-Factor App
Methodology for building cloud native applications:

| Factor | Description |
|--------|-------------|
| **1. Codebase** | One codebase tracked in version control |
| **2. Dependencies** | Explicitly declare and isolate dependencies |
| **3. Config** | Store config in environment variables |
| **4. Backing Services** | Treat backing services as attached resources |
| **5. Build, Release, Run** | Strictly separate build and run stages |
| **6. Processes** | Execute the app as stateless processes |
| **7. Port Binding** | Export services via port binding |
| **8. Concurrency** | Scale out via the process model |
| **9. Disposability** | Fast startup and graceful shutdown |
| **10. Dev/Prod Parity** | Keep development and production similar |
| **11. Logs** | Treat logs as event streams |
| **12. Admin Processes** | Run admin tasks as one-off processes |

---

## 3.4 API Gateways

### What is an API Gateway?
- **Definition**: Single entry point for all client requests
- **Purpose**: Routes requests to appropriate microservices

### API Gateway Functions
- **Request Routing**: Direct requests to backend services
- **Authentication/Authorization**: Verify user identity
- **Rate Limiting**: Control request rates
- **Load Balancing**: Distribute traffic
- **Caching**: Reduce backend load
- **Request/Response Transformation**: Modify payloads
- **Logging and Monitoring**: Track API usage

### API Gateway vs Ingress

| Feature | API Gateway | Kubernetes Ingress |
|---------|-------------|-------------------|
| **Scope** | API management | HTTP routing |
| **Features** | Full API lifecycle | Basic L7 routing |
| **Authentication** | Advanced | Limited |
| **Rate Limiting** | Built-in | Requires annotations |
| **Examples** | Kong, Ambassador | nginx-ingress, Traefik |

### Popular API Gateways
- **Kong**: Open source, plugin ecosystem
- **Ambassador**: Kubernetes-native, Envoy-based
- **AWS API Gateway**: Managed service
- **Apigee**: Google Cloud API management
- **NGINX**: Can act as API gateway

---

## 3.5 Serverless Computing

### What is Serverless?
- **Definition**: Cloud execution model where the provider manages infrastructure
- **Characteristics**:
  - No server management
  - Automatic scaling
  - Pay-per-use pricing
  - Event-driven execution

### Types of Serverless

#### Function as a Service (FaaS)
- Run individual functions in response to events
- Examples: AWS Lambda, Google Cloud Functions, Azure Functions
- Kubernetes: Knative, OpenFaaS, Kubeless

#### Backend as a Service (BaaS)
- Pre-built backend services
- Examples: Firebase, Auth0, AWS Cognito

### Serverless Benefits
- ✅ No infrastructure management
- ✅ Automatic scaling (including to zero)
- ✅ Pay only for actual usage
- ✅ Focus on code, not operations
- ✅ Reduced operational complexity

### Serverless Challenges
- ❌ Cold start latency
- ❌ Vendor lock-in
- ❌ Limited execution duration
- ❌ Debugging and testing complexity
- ❌ State management challenges

### Knative
- **Purpose**: Kubernetes-native serverless platform
- **Status**: CNCF Incubating
- **Components**:
  - **Serving**: Deploy and auto-scale serverless workloads
  - **Eventing**: Event-driven architectures

---

## 3.6 Autoscaling

### Types of Autoscaling in Kubernetes

#### Horizontal Pod Autoscaler (HPA)
- **Purpose**: Scale number of Pod replicas
- **Based on**: CPU, memory, or custom metrics
- **How it works**: Adjusts replica count in Deployment/ReplicaSet

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

#### Vertical Pod Autoscaler (VPA)
- **Purpose**: Adjust CPU/memory requests for Pods
- **Based on**: Historical resource usage
- **Note**: May require Pod restart

#### Cluster Autoscaler
- **Purpose**: Scale number of nodes in cluster
- **When**: Pods are pending due to insufficient resources
- **Cloud Providers**: Works with AWS, GCP, Azure, etc.

### KEDA (Kubernetes Event-driven Autoscaling)
- **Purpose**: Event-driven autoscaling for Kubernetes
- **Status**: CNCF Graduated
- **Features**:
  - Scale based on external events (queues, databases, etc.)
  - Scale to zero capability
  - Many built-in scalers (Kafka, RabbitMQ, Azure queues, etc.)

---

## 3.7 Cloud Native Storage

### Storage Types

#### Ephemeral Storage
- **Lifecycle**: Same as Pod
- **Use Cases**: Temporary files, caches
- **Types**: emptyDir

#### Persistent Storage
- **Lifecycle**: Independent of Pod
- **Use Cases**: Databases, stateful applications
- **Components**: PV, PVC, StorageClass

### Cloud Native Storage Solutions

| Solution | Type | Description |
|----------|------|-------------|
| **Rook** | CNCF Graduated | Storage orchestration for Kubernetes |
| **Longhorn** | CNCF Incubating | Cloud native distributed block storage |
| **Ceph** | Open Source | Unified storage (block, file, object) |
| **MinIO** | Open Source | S3-compatible object storage |
| **OpenEBS** | CNCF Sandbox | Container native storage |

### Container Storage Interface (CSI)
- **Purpose**: Standard interface for storage plugins
- **Benefits**:
  - Storage vendor independence
  - Dynamic provisioning
  - Snapshots and cloning

---

## 3.8 Infrastructure as Code (IaC)

### What is IaC?
- **Definition**: Managing infrastructure through machine-readable files
- **Benefits**:
  - Version control for infrastructure
  - Reproducible environments
  - Automation and consistency
  - Self-documenting infrastructure

### IaC Tools

| Tool | Type | Description |
|------|------|-------------|
| **Terraform** | Declarative | Multi-cloud infrastructure provisioning |
| **Pulumi** | Imperative/Declarative | IaC using general programming languages |
| **Ansible** | Configuration Management | Automation and configuration |
| **Crossplane** | Kubernetes-native | Manage cloud resources via Kubernetes |
| **AWS CloudFormation** | Declarative | AWS-specific IaC |

### GitOps for Infrastructure
- **Concept**: Manage infrastructure declaratively in Git
- **Tools**: ArgoCD, Flux, Terraform with Git
- **Benefits**: Audit trail, review process, automation

---

## 3.9 Cloud Native Security

### Security Principles

#### Defense in Depth
Multiple layers of security controls:
1. **Network**: Network policies, firewalls
2. **Platform**: RBAC, Pod Security Standards
3. **Application**: Secure coding, input validation
4. **Data**: Encryption at rest and in transit

#### Least Privilege
- Grant minimum necessary permissions
- Use RBAC effectively
- Avoid running as root

### 4C's of Cloud Native Security

```
┌───────────────────────────────────────────────┐
│                    CODE                       │
│  (Application security, dependencies)         │
├───────────────────────────────────────────────┤
│                  CONTAINER                    │
│  (Image scanning, runtime security)           │
├───────────────────────────────────────────────┤
│                   CLUSTER                     │
│  (RBAC, Pod Security, Network Policies)       │
├───────────────────────────────────────────────┤
│                    CLOUD                      │
│  (Cloud provider security, IAM)               │
└───────────────────────────────────────────────┘
```

### Security Tools

| Tool | Purpose | CNCF Status |
|------|---------|-------------|
| **Falco** | Runtime security | Graduated |
| **Open Policy Agent** | Policy engine | Graduated |
| **Trivy** | Vulnerability scanning | - |
| **Kyverno** | Policy management | Incubating |
| **cert-manager** | Certificate management | Incubating |

---

## 3.10 Cloud Providers and Managed Kubernetes

### Major Cloud Providers

| Provider | Managed Kubernetes | Region |
|----------|-------------------|--------|
| **AWS** | Amazon EKS | Global |
| **Google Cloud** | Google GKE | Global |
| **Microsoft Azure** | Azure AKS | Global |
| **IBM Cloud** | IBM Kubernetes Service | Global |
| **DigitalOcean** | DOKS | Limited |
| **Alibaba Cloud** | ACK | Asia-focused |

### Managed vs Self-Managed Kubernetes

| Aspect | Managed | Self-Managed |
|--------|---------|--------------|
| **Control Plane** | Provider managed | You manage |
| **Upgrades** | Easier | Manual |
| **Cost** | Higher | Lower (just compute) |
| **Customization** | Limited | Full control |
| **Expertise Required** | Less | More |

### Multi-Cloud and Hybrid Cloud

**Multi-Cloud:**
- Using multiple cloud providers
- Avoid vendor lock-in
- Best-of-breed services

**Hybrid Cloud:**
- Combination of public cloud and on-premises
- Data sovereignty requirements
- Legacy system integration

---

## Key Exam Tips for This Domain

1. **Understand CNCF project maturity levels** (Sandbox, Incubating, Graduated)
2. **Know key graduated projects** and their purposes
3. **Understand microservices** vs monolithic architecture
4. **Know the Twelve-Factor App** principles
5. **Understand serverless** concepts and Knative
6. **Know autoscaling types** (HPA, VPA, Cluster Autoscaler)
7. **Understand the 4C's** of cloud native security
8. **Know IaC concepts** and popular tools

---

## Practice Questions

1. What are the three CNCF project maturity levels?
   - Answer: Sandbox, Incubating, Graduated

2. Which CNCF project is used for distributed tracing?
   - Answer: Jaeger

3. What is the purpose of a serverless architecture?
   - Answer: Run applications without managing servers, with automatic scaling and pay-per-use pricing

4. What does HPA stand for in Kubernetes?
   - Answer: Horizontal Pod Autoscaler

5. Name three characteristics of microservices architecture.
   - Answer: Any three of: Single responsibility, independently deployable, decentralized data, technology diversity, fault isolation, scalable

6. What is the Kubernetes-native serverless platform?
   - Answer: Knative

7. What are the 4C's of Cloud Native Security?
   - Answer: Code, Container, Cluster, Cloud

8. Which tool is used for policy-as-code in Kubernetes?
   - Answer: Open Policy Agent (OPA) or Kyverno

---

## Additional Resources

- [CNCF Cloud Native Definition](https://github.com/cncf/toc/blob/main/DEFINITION.md)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Twelve-Factor App](https://12factor.net/)
- [Knative Documentation](https://knative.dev/docs/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)