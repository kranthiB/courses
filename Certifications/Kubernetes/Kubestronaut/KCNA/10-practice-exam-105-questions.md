# KCNA Practice Exam - 100+ Questions with Answers

## Instructions
- The real exam has 60 questions in 90 minutes
- Passing score: 75% (45 correct)
- For 100%: You need to master ALL concepts
- Practice these questions multiple times until you get 100%

---

# DOMAIN 1: KUBERNETES FUNDAMENTALS (46%)

## Control Plane Components

### Q1. Which component is the front-end for the Kubernetes control plane?
- A) etcd
- B) kube-scheduler
- C) kube-apiserver ✓
- D) kube-controller-manager

**Explanation**: The kube-apiserver exposes the Kubernetes API and is the front-end for the control plane. All communication goes through it.

---

### Q2. Where is all cluster data stored in Kubernetes?
- A) kube-apiserver
- B) etcd ✓
- C) kubelet
- D) kube-scheduler

**Explanation**: etcd is the consistent and highly-available key-value store used as Kubernetes' backing store for all cluster data.

---

### Q3. Which component is responsible for assigning Pods to Nodes?
- A) kubelet
- B) kube-controller-manager
- C) kube-scheduler ✓
- D) kube-proxy

**Explanation**: The kube-scheduler watches for newly created Pods with no assigned node and selects a node for them to run on.

---

### Q4. Which controller is NOT part of kube-controller-manager?
- A) Node Controller
- B) Replication Controller
- C) Ingress Controller ✓
- D) Endpoints Controller

**Explanation**: Ingress Controller is a separate component, not part of kube-controller-manager. It's typically deployed as a Pod.

---

### Q5. What consensus algorithm does etcd use?
- A) Paxos
- B) Raft ✓
- C) PBFT
- D) Zab

**Explanation**: etcd uses the Raft consensus algorithm for distributed consistency.

---

## Worker Node Components

### Q6. Which component runs on each node and ensures containers are running in Pods?
- A) kube-proxy
- B) kubelet ✓
- C) container runtime
- D) kube-scheduler

**Explanation**: The kubelet is the primary node agent that ensures containers described in PodSpecs are running and healthy.

---

### Q7. What is the primary function of kube-proxy?
- A) Run containers
- B) Schedule Pods
- C) Maintain network rules ✓
- D) Store cluster data

**Explanation**: kube-proxy maintains network rules on nodes that allow network communication to Pods from inside or outside the cluster.

---

### Q8. Which of the following is a valid container runtime for Kubernetes?
- A) containerd ✓
- B) kubectl
- C) kubeadm
- D) kube-proxy

**Explanation**: containerd is a container runtime that implements the Container Runtime Interface (CRI). Others include CRI-O.

---

### Q9. The kubelet communicates with container runtimes using which interface?
- A) CNI
- B) CSI
- C) CRI ✓
- D) OCI

**Explanation**: CRI (Container Runtime Interface) is the standard interface between kubelet and container runtimes.

---

## Kubernetes Objects

### Q10. What is the smallest deployable unit in Kubernetes?
- A) Container
- B) Pod ✓
- C) Deployment
- D) ReplicaSet

**Explanation**: A Pod is the smallest deployable unit in Kubernetes, containing one or more containers.

---

### Q11. Which resource ensures a specified number of Pod replicas are running?
- A) Deployment
- B) ReplicaSet ✓
- C) DaemonSet
- D) StatefulSet

**Explanation**: ReplicaSet ensures that a specified number of pod replicas are running at any given time.

---

### Q12. What is the recommended way to manage Pods in production?
- A) Create Pods directly
- B) Use ReplicaSets directly
- C) Use Deployments ✓
- D) Use Jobs

**Explanation**: Deployments provide declarative updates for Pods and ReplicaSets, with features like rolling updates and rollbacks.

---

### Q13. Which Kubernetes object ensures a Pod runs on ALL nodes?
- A) ReplicaSet
- B) Deployment
- C) DaemonSet ✓
- D) StatefulSet

**Explanation**: DaemonSet ensures that all (or some) nodes run a copy of a Pod, commonly used for logging and monitoring agents.

---

### Q14. Which object is best suited for running a database that requires stable network identity?
- A) Deployment
- B) ReplicaSet
- C) DaemonSet
- D) StatefulSet ✓

**Explanation**: StatefulSets provide stable, unique network identifiers and stable, persistent storage - ideal for databases.

---

### Q15. What is a ConfigMap used for?
- A) Store sensitive data
- B) Store non-confidential configuration data ✓
- C) Define network policies
- D) Manage storage

**Explanation**: ConfigMaps store non-confidential configuration data in key-value pairs.

---

### Q16. How are Secrets stored in Kubernetes by default?
- A) Encrypted
- B) Base64 encoded ✓
- C) Plain text
- D) Hashed

**Explanation**: Secrets are base64 encoded by default, NOT encrypted. Encryption at rest must be configured separately.

---

### Q17. Which namespace contains Kubernetes system components?
- A) default
- B) kube-system ✓
- C) kube-public
- D) kube-node-lease

**Explanation**: kube-system namespace contains objects created by the Kubernetes system.

---

### Q18. What does a Job resource do when the Pod completes successfully?
- A) Restarts the Pod
- B) Deletes the Pod immediately
- C) Keeps the Pod for log inspection ✓
- D) Creates a new Pod

**Explanation**: When a Job completes, the Pod is kept (not deleted) so you can check its logs. Use TTL controller for automatic cleanup.

---

## Services

### Q19. What is the default Service type in Kubernetes?
- A) NodePort
- B) LoadBalancer
- C) ClusterIP ✓
- D) ExternalName

**Explanation**: ClusterIP is the default Service type, which exposes the Service on a cluster-internal IP.

---

### Q20. Which Service type exposes a service on each Node's IP at a static port?
- A) ClusterIP
- B) NodePort ✓
- C) LoadBalancer
- D) ExternalName

**Explanation**: NodePort exposes the Service on each Node's IP at a static port in the range 30000-32767.

---

### Q21. Which Service type provisions a cloud provider's load balancer?
- A) ClusterIP
- B) NodePort
- C) LoadBalancer ✓
- D) ExternalName

**Explanation**: LoadBalancer exposes the Service externally using a cloud provider's load balancer.

---

### Q22. What is a Headless Service?
- A) A Service with no selector
- B) A Service with ClusterIP set to None ✓
- C) A Service with no endpoints
- D) A Service with no ports

**Explanation**: A Headless Service has ClusterIP: None, and returns Pod IPs directly instead of a single cluster IP.

---

### Q23. How does a Service identify which Pods to route traffic to?
- A) By Pod name
- B) By namespace
- C) By label selector ✓
- D) By IP address

**Explanation**: Services use label selectors to identify the set of Pods to route traffic to.

---

## Labels and Selectors

### Q24. What is the maximum length of a label key?
- A) 32 characters
- B) 63 characters ✓
- C) 128 characters
- D) 256 characters

**Explanation**: Label keys can have a maximum of 63 characters (253 for the prefix if used).

---

### Q25. Which selector type supports "in" and "notin" operators?
- A) Equality-based
- B) Set-based ✓
- C) Both
- D) Neither

**Explanation**: Set-based selectors support operators: in, notin, exists, and !exists.

---

## kubectl Commands

### Q26. Which command shows all pods in all namespaces?
- A) kubectl get pods
- B) kubectl get pods --all
- C) kubectl get pods -A ✓
- D) kubectl get pods -n all

**Explanation**: `kubectl get pods -A` or `kubectl get pods --all-namespaces` lists pods across all namespaces.

---

### Q27. Which command creates a deployment named "web" with the nginx image?
- A) kubectl run web --image=nginx
- B) kubectl create deployment web --image=nginx ✓
- C) kubectl deploy web --image=nginx
- D) kubectl new deployment web --image=nginx

**Explanation**: `kubectl create deployment <name> --image=<image>` is the correct imperative command.

---

### Q28. How do you scale a deployment named "web" to 5 replicas?
- A) kubectl replicas deployment web 5
- B) kubectl scale deployment web --replicas=5 ✓
- C) kubectl set replicas deployment web 5
- D) kubectl update deployment web --replicas=5

**Explanation**: `kubectl scale deployment <name> --replicas=<count>` scales the deployment.

---

### Q29. Which command shows detailed information about a Pod?
- A) kubectl get pod <name> -v
- B) kubectl describe pod <name> ✓
- C) kubectl info pod <name>
- D) kubectl show pod <name>

**Explanation**: `kubectl describe` shows detailed information about a resource including events.

---

### Q30. Which command streams logs from a Pod in real-time?
- A) kubectl logs <pod> --stream
- B) kubectl logs <pod> -f ✓
- C) kubectl logs <pod> --live
- D) kubectl logs <pod> -r

**Explanation**: The `-f` flag (follow) streams logs in real-time.

---

## Scheduling

### Q31. What happens when a Pod cannot be scheduled due to insufficient resources?
- A) Pod is deleted
- B) Pod remains in Pending state ✓
- C) Pod is scheduled anyway
- D) Pod is moved to another cluster

**Explanation**: When no node has sufficient resources, the Pod remains in Pending state until resources become available.

---

### Q32. What is the effect of the "NoSchedule" taint?
- A) Evicts existing Pods
- B) Prevents new Pods from scheduling ✓
- C) Prefers not to schedule but allows
- D) Has no effect

**Explanation**: NoSchedule prevents new Pods without matching tolerations from being scheduled on the node.

---

### Q33. Which taint effect evicts existing Pods that don't tolerate the taint?
- A) NoSchedule
- B) PreferNoSchedule
- C) NoExecute ✓
- D) Evict

**Explanation**: NoExecute evicts existing Pods that don't have a matching toleration.

---

### Q34. What is node affinity?
- A) Tainting nodes
- B) Constraining which nodes a Pod can be scheduled on ✓
- C) Spreading Pods across nodes
- D) Evicting Pods from nodes

**Explanation**: Node affinity allows you to constrain which nodes your Pod can be scheduled on based on node labels.

---

## Probes

### Q35. Which probe determines if a container should be restarted?
- A) Readiness probe
- B) Liveness probe ✓
- C) Startup probe
- D) Health probe

**Explanation**: If a liveness probe fails, the kubelet kills the container and restarts it.

---

### Q36. Which probe determines if a container should receive traffic?
- A) Liveness probe
- B) Readiness probe ✓
- C) Startup probe
- D) Traffic probe

**Explanation**: If a readiness probe fails, the Pod is removed from Service endpoints (no traffic).

---

### Q37. Which probe type is best for slow-starting containers?
- A) Liveness probe
- B) Readiness probe
- C) Startup probe ✓
- D) Init probe

**Explanation**: Startup probes disable liveness and readiness checks until the application has started.

---

### Q38. Which probe method makes an HTTP request to the container?
- A) exec
- B) tcpSocket
- C) httpGet ✓
- D) grpc

**Explanation**: httpGet performs an HTTP GET request against the Pod's IP.

---

---

# DOMAIN 2: CONTAINER ORCHESTRATION (22%)

## Container Runtime

### Q39. What does OCI stand for?
- A) Open Cloud Initiative
- B) Open Container Initiative ✓
- C) Orchestration Container Interface
- D) Open Compute Infrastructure

**Explanation**: OCI (Open Container Initiative) defines standards for container formats and runtimes.

---

### Q40. Which is NOT a container runtime?
- A) containerd
- B) CRI-O
- C) Docker
- D) kubectl ✓

**Explanation**: kubectl is a CLI tool, not a container runtime. containerd, CRI-O, and Docker are runtimes.

---

### Q41. What replaced Docker as the default runtime in Kubernetes 1.24+?
- A) CRI-O
- B) containerd ✓
- C) runc
- D) podman

**Explanation**: containerd became the default runtime after Kubernetes removed dockershim in v1.24.

---

## Networking

### Q42. What does CNI stand for?
- A) Container Network Infrastructure
- B) Container Network Interface ✓
- C) Cloud Native Interface
- D) Cluster Network Integration

**Explanation**: CNI (Container Network Interface) is a specification for configuring network interfaces in Linux containers.

---

### Q43. Which CNI plugin uses eBPF technology?
- A) Flannel
- B) Calico
- C) Cilium ✓
- D) Weave

**Explanation**: Cilium uses eBPF (extended Berkeley Packet Filter) for networking, security, and observability.

---

### Q44. What is the default DNS server in Kubernetes?
- A) kube-dns
- B) CoreDNS ✓
- C) BIND
- D) dnsmasq

**Explanation**: CoreDNS is the default DNS server in Kubernetes since version 1.13.

---

### Q45. What does a NetworkPolicy control?
- A) Node-to-node traffic
- B) Pod-to-pod traffic ✓
- C) Service-to-service traffic
- D) Cluster-to-cluster traffic

**Explanation**: NetworkPolicies control traffic flow at the Pod level (Layer 3/4).

---

### Q46. By default, what traffic is allowed between Pods?
- A) No traffic
- B) Only within namespace
- C) All traffic ✓
- D) Only TCP traffic

**Explanation**: By default, all traffic is allowed between Pods. NetworkPolicies restrict this.

---

## Security

### Q47. What does RBAC stand for?
- A) Resource Based Access Configuration
- B) Role Based Access Control ✓
- C) Rule Based Access Control
- D) Resource Based Action Control

**Explanation**: RBAC (Role Based Access Control) regulates access to resources based on user roles.

---

### Q48. Which RBAC resource is namespace-scoped?
- A) ClusterRole
- B) ClusterRoleBinding
- C) Role ✓
- D) ServiceAccount

**Explanation**: Role is namespace-scoped, while ClusterRole is cluster-scoped.

---

### Q49. What are the three Pod Security Standards levels?
- A) Low, Medium, High
- B) Privileged, Baseline, Restricted ✓
- C) Open, Default, Secure
- D) Basic, Standard, Advanced

**Explanation**: Pod Security Standards define three levels: Privileged, Baseline, and Restricted.

---

### Q50. Which securityContext setting prevents privilege escalation?
- A) runAsNonRoot
- B) readOnlyRootFilesystem
- C) allowPrivilegeEscalation: false ✓
- D) privileged: false

**Explanation**: `allowPrivilegeEscalation: false` prevents a process from gaining more privileges than its parent.

---

## Service Mesh

### Q51. What is the primary function of a service mesh?
- A) Container orchestration
- B) Service-to-service communication ✓
- C) Pod scheduling
- D) Log aggregation

**Explanation**: A service mesh handles service-to-service communication, including traffic management, security, and observability.

---

### Q52. Which is a CNCF graduated service mesh?
- A) Consul
- B) Istio ✓
- C) AWS App Mesh
- D) Kuma

**Explanation**: Both Istio and Linkerd are CNCF graduated service mesh projects.

---

### Q53. What proxy does Istio use for its data plane?
- A) NGINX
- B) HAProxy
- C) Envoy ✓
- D) Traefik

**Explanation**: Istio uses Envoy as its sidecar proxy in the data plane.

---

### Q54. What is mTLS in service mesh context?
- A) Multi-tenant TLS
- B) Mutual TLS ✓
- C) Managed TLS
- D) Mesh TLS

**Explanation**: mTLS (Mutual TLS) means both client and server authenticate each other with certificates.

---

## Storage

### Q55. What does PV stand for in Kubernetes storage?
- A) Pod Volume
- B) Persistent Volume ✓
- C) Private Volume
- D) Provision Volume

**Explanation**: PV (Persistent Volume) is a cluster-level storage resource.

---

### Q56. What does PVC stand for?
- A) Pod Volume Claim
- B) Persistent Volume Claim ✓
- C) Private Volume Claim
- D) Provision Volume Claim

**Explanation**: PVC (Persistent Volume Claim) is a request for storage by a user.

---

### Q57. Which access mode allows the volume to be mounted as read-write by multiple nodes?
- A) ReadWriteOnce (RWO)
- B) ReadOnlyMany (ROX)
- C) ReadWriteMany (RWX) ✓
- D) ReadWriteOncePod (RWOP)

**Explanation**: RWX (ReadWriteMany) allows the volume to be mounted as read-write by many nodes.

---

### Q58. What does CSI stand for?
- A) Container Storage Instance
- B) Container Storage Interface ✓
- C) Cloud Storage Integration
- D) Cluster Storage Infrastructure

**Explanation**: CSI (Container Storage Interface) is the standard for exposing storage systems to Kubernetes.

---

### Q59. Which reclaim policy keeps the PV data after PVC deletion?
- A) Delete
- B) Recycle
- C) Retain ✓
- D) Preserve

**Explanation**: Retain keeps the PV and its data after the PVC is deleted (manual reclamation).

---

---

# DOMAIN 3: CLOUD NATIVE ARCHITECTURE (16%)

## CNCF

### Q60. What are the three CNCF project maturity levels?
- A) Alpha, Beta, Stable
- B) Sandbox, Incubating, Graduated ✓
- C) Experimental, Testing, Production
- D) Development, Staging, Production

**Explanation**: CNCF projects progress through Sandbox → Incubating → Graduated maturity levels.

---

### Q61. Which is NOT a CNCF graduated project?
- A) Kubernetes
- B) Prometheus
- C) Docker ✓
- D) Helm

**Explanation**: Docker is not a CNCF project. Kubernetes, Prometheus, and Helm are graduated projects.

---

### Q62. What CNCF project is used for distributed tracing?
- A) Prometheus
- B) Fluentd
- C) Jaeger ✓
- D) Grafana

**Explanation**: Jaeger is the CNCF graduated project for distributed tracing.

---

### Q63. Which project provides a unified observability framework?
- A) Prometheus
- B) OpenTelemetry ✓
- C) Grafana
- D) Datadog

**Explanation**: OpenTelemetry provides vendor-neutral APIs and tools for metrics, logs, and traces.

---

## Microservices

### Q64. What is a key characteristic of microservices architecture?
- A) Single codebase for all services
- B) Shared database
- C) Independently deployable services ✓
- D) Synchronous communication only

**Explanation**: Microservices are independently deployable, loosely coupled services.

---

### Q65. Which is NOT a benefit of microservices?
- A) Independent scaling
- B) Technology flexibility
- C) Simpler debugging ✓
- D) Fault isolation

**Explanation**: Debugging is actually MORE complex in microservices due to distributed nature.

---

### Q66. How many factors are in the Twelve-Factor App methodology?
- A) 10
- B) 12 ✓
- C) 15
- D) 20

**Explanation**: The Twelve-Factor App methodology has 12 factors for building cloud-native applications.

---

### Q67. According to Twelve-Factor, where should configuration be stored?
- A) In code
- B) In config files
- C) In environment variables ✓
- D) In a database

**Explanation**: Factor III states: "Store config in the environment" (environment variables).

---

## Serverless

### Q68. What is a key characteristic of serverless computing?
- A) You manage servers
- B) Fixed pricing
- C) Automatic scaling to zero ✓
- D) Always running

**Explanation**: Serverless can scale to zero when not in use, and you pay only for actual execution.

---

### Q69. What does FaaS stand for?
- A) Framework as a Service
- B) Function as a Service ✓
- C) Feature as a Service
- D) Firewall as a Service

**Explanation**: FaaS (Function as a Service) allows running individual functions in response to events.

---

### Q70. Which is the Kubernetes-native serverless platform?
- A) AWS Lambda
- B) Azure Functions
- C) Knative ✓
- D) OpenFaaS

**Explanation**: Knative is the CNCF project for serverless workloads on Kubernetes.

---

## Autoscaling

### Q71. What does HPA scale?
- A) Nodes
- B) Pods ✓
- C) Clusters
- D) Namespaces

**Explanation**: HPA (Horizontal Pod Autoscaler) scales the number of Pod replicas.

---

### Q72. What does VPA adjust?
- A) Number of pods
- B) Number of nodes
- C) Resource requests/limits ✓
- D) Network bandwidth

**Explanation**: VPA (Vertical Pod Autoscaler) adjusts CPU and memory requests/limits.

---

### Q73. Which component scales cluster nodes?
- A) HPA
- B) VPA
- C) Cluster Autoscaler ✓
- D) KEDA

**Explanation**: Cluster Autoscaler automatically adjusts the number of nodes in the cluster.

---

### Q74. What does KEDA provide?
- A) Container runtime
- B) Event-driven autoscaling ✓
- C) Service mesh
- D) Logging

**Explanation**: KEDA (Kubernetes Event-driven Autoscaling) scales based on external events.

---

## Security

### Q75. What are the 4C's of Cloud Native Security?
- A) Code, Container, Cluster, Cloud ✓
- B) Code, Compile, Container, Cloud
- C) Config, Container, Cluster, Cloud
- D) Code, Container, Cluster, Customer

**Explanation**: The 4C's are Code, Container, Cluster, and Cloud - layers of security.

---

### Q76. Which CNCF project is used for runtime security?
- A) OPA
- B) Falco ✓
- C) Trivy
- D) Kyverno

**Explanation**: Falco is the CNCF graduated project for runtime security and threat detection.

---

---

# DOMAIN 4: CLOUD NATIVE OBSERVABILITY (8%)

### Q77. What are the three pillars of observability?
- A) Metrics, Logs, Traces ✓
- B) Metrics, Events, Alerts
- C) Logs, Alerts, Dashboards
- D) Monitoring, Logging, Tracing

**Explanation**: The three pillars are Metrics (numerical data), Logs (event records), and Traces (request paths).

---

### Q78. Which metric type only increases?
- A) Gauge
- B) Counter ✓
- C) Histogram
- D) Summary

**Explanation**: A Counter is a cumulative metric that only increases (resets on restart).

---

### Q79. What query language does Prometheus use?
- A) SQL
- B) PromQL ✓
- C) LogQL
- D) GraphQL

**Explanation**: Prometheus uses PromQL (Prometheus Query Language) for querying metrics.

---

### Q80. How does Prometheus collect metrics?
- A) Push model
- B) Pull model ✓
- C) Both push and pull
- D) Event-driven

**Explanation**: Prometheus uses a pull model - it scrapes metrics from target endpoints.

---

### Q81. What is the difference between Fluentd and Fluent Bit?
- A) Fluentd is newer
- B) Fluent Bit is more feature-rich
- C) Fluent Bit is lighter weight ✓
- D) They are the same

**Explanation**: Fluent Bit is lightweight (C-based), Fluentd is full-featured (Ruby-based).

---

### Q82. What does Grafana Loki index?
- A) Full log text
- B) Labels only ✓
- C) Timestamps
- D) Nothing

**Explanation**: Loki only indexes labels/metadata, not the full log content, making it cost-effective.

---

### Q83. What are the Google SRE Golden Signals?
- A) CPU, Memory, Disk, Network
- B) Latency, Traffic, Errors, Saturation ✓
- C) Requests, Errors, Duration, Queue
- D) Availability, Reliability, Performance, Security

**Explanation**: The four golden signals are Latency, Traffic, Errors, and Saturation.

---

### Q84. What does SLO stand for?
- A) Service Level Objective ✓
- B) Service Level Output
- C) System Level Objective
- D) Standard Level Objective

**Explanation**: SLO (Service Level Objective) is a target value for a service level indicator.

---

### Q85. Which tool is used for distributed tracing in CNCF?
- A) Prometheus
- B) Jaeger ✓
- C) Fluentd
- D) Grafana

**Explanation**: Jaeger is the CNCF graduated project for distributed tracing.

---

### Q86. What is a trace composed of?
- A) Metrics
- B) Spans ✓
- C) Logs
- D) Events

**Explanation**: A trace is composed of spans, where each span represents an operation.

---

---

# DOMAIN 5: CLOUD NATIVE APPLICATION DELIVERY (8%)

### Q87. What is the difference between Continuous Delivery and Continuous Deployment?
- A) No difference
- B) Delivery requires manual approval for production ✓
- C) Deployment requires manual approval
- D) Delivery is faster

**Explanation**: Continuous Delivery has a manual approval step before production; Deployment is fully automated.

---

### Q88. What is the source of truth in GitOps?
- A) Kubernetes cluster
- B) CI/CD pipeline
- C) Git repository ✓
- D) Container registry

**Explanation**: In GitOps, Git is the single source of truth for declarative infrastructure and applications.

---

### Q89. Which are CNCF graduated GitOps tools? (Select TWO)
- A) Jenkins
- B) Argo CD ✓
- C) Flux CD ✓
- D) Spinnaker

**Explanation**: Both Argo CD and Flux CD are CNCF graduated GitOps tools.

---

### Q90. What deployment strategy gradually shifts traffic to a new version?
- A) Blue-Green
- B) Recreate
- C) Canary ✓
- D) Rolling

**Explanation**: Canary deployments gradually shift traffic (e.g., 10%, 30%, 50%, 100%) to the new version.

---

### Q91. What deployment strategy runs two identical environments?
- A) Canary
- B) Blue-Green ✓
- C) Rolling
- D) Recreate

**Explanation**: Blue-Green maintains two environments and switches traffic between them.

---

### Q92. What is the default deployment strategy in Kubernetes?
- A) Blue-Green
- B) Canary
- C) Rolling Update ✓
- D) Recreate

**Explanation**: RollingUpdate is the default strategy, gradually replacing old Pods with new ones.

---

### Q93. What is a Helm Chart?
- A) A monitoring dashboard
- B) A package of Kubernetes resources ✓
- C) A deployment strategy
- D) A container image

**Explanation**: A Helm Chart is a package containing Kubernetes resource definitions.

---

### Q94. What file contains default values in a Helm chart?
- A) Chart.yaml
- B) values.yaml ✓
- C) config.yaml
- D) default.yaml

**Explanation**: values.yaml contains default configuration values for a chart.

---

### Q95. What command applies Kustomize overlays?
- A) kubectl kustomize apply
- B) kubectl apply -k ✓
- C) kustomize apply
- D) kubectl apply --kustomize

**Explanation**: `kubectl apply -k <directory>` applies Kustomize configurations.

---

### Q96. What is the main difference between Helm and Kustomize?
- A) Helm is newer
- B) Helm uses templates, Kustomize uses overlays ✓
- C) Kustomize uses templates
- D) They are the same

**Explanation**: Helm uses Go templates; Kustomize uses base + overlay patching approach.

---

### Q97. Which is a Kubernetes-native CI/CD framework?
- A) Jenkins
- B) GitHub Actions
- C) Tekton ✓
- D) CircleCI

**Explanation**: Tekton is a Kubernetes-native CI/CD framework from the CD Foundation.

---

---

# BONUS TRICKY QUESTIONS

### Q98. Can a Pod have multiple containers?
- A) No, only one container per Pod
- B) Yes, but only init containers
- C) Yes, multiple containers can share resources ✓
- D) Yes, but they must be identical

**Explanation**: Pods can have multiple containers that share network and storage (sidecar pattern).

---

### Q99. What happens to a Pod when its node fails?
- A) Pod is automatically rescheduled
- B) Pod is marked for deletion ✓
- C) Pod continues running
- D) Pod is immediately deleted

**Explanation**: When a node fails, Pods are marked for deletion. The controller creates replacements.

---

### Q100. Which component does NOT run on worker nodes?
- A) kubelet
- B) kube-proxy
- C) Container runtime
- D) kube-scheduler ✓

**Explanation**: kube-scheduler runs on control plane nodes, not worker nodes.

---

### Q101. What is the purpose of an Init Container?
- A) Run alongside app containers
- B) Run before app containers start ✓
- C) Run after app containers stop
- D) Replace app containers

**Explanation**: Init containers run and complete before app containers start.

---

### Q102. Can you update a ConfigMap and have Pods automatically reload?
- A) Yes, always
- B) No, Pods must be restarted ✓
- C) Only with Secrets
- D) Only with environment variables

**Explanation**: ConfigMap changes don't automatically restart Pods. Mounted volumes update eventually, but env vars require restart.

---

### Q103. What is an Ephemeral Container used for?
- A) Running production workloads
- B) Debugging running Pods ✓
- C) Init processes
- D) Sidecar services

**Explanation**: Ephemeral containers are added to running Pods for debugging purposes.

---

### Q104. Which is TRUE about Kubernetes Secrets?
- A) They are encrypted by default
- B) They are base64 encoded by default ✓
- C) They cannot be mounted as files
- D) They are more secure than ConfigMaps by default

**Explanation**: Secrets are base64 encoded (not encrypted) by default. Enable encryption at rest for security.

---

### Q105. What does "kubectl drain" do?
- A) Deletes a node
- B) Removes all pods and marks node unschedulable ✓
- C) Empties a namespace
- D) Clears logs

**Explanation**: `kubectl drain` safely evicts pods and marks the node as unschedulable (for maintenance).

---

## Scoring Guide

| Score | Readiness |
|-------|-----------|
| 90-105 | Ready for 100%! |
| 75-89 | Ready to pass, review weak areas |
| 60-74 | Need more study |
| Below 60 | Significant preparation needed |

**Target: Get 100% on this practice exam before taking the real KCNA.**