# CKA Study Guide: Services and Networking (20%)

## Domain Overview

The Services and Networking domain covers Kubernetes networking concepts including Pod connectivity, Network Policies, Service types, Gateway API, Ingress controllers, and CoreDNS configuration.

---

## 1. Understand Connectivity Between Pods

### 1.1 Kubernetes Networking Model

Kubernetes imposes the following fundamental networking requirements:

1. **Every Pod gets its own IP address**
2. **Pods can communicate with all other Pods without NAT**
3. **Nodes can communicate with all Pods without NAT**
4. **The IP that a Pod sees itself as is the same IP others see it as**

### 1.2 Pod-to-Pod Communication

#### Same Node

Pods on the same node communicate through the virtual ethernet bridge.

```bash
# Check pod IPs
kubectl get pods -o wide

# Test connectivity between pods
kubectl exec <pod1> -- ping <pod2-ip>
kubectl exec <pod1> -- curl <pod2-ip>:<port>
```

#### Different Nodes

The CNI plugin handles cross-node Pod communication using overlay networks or direct routing.

### 1.3 Pod Network Inspection

```bash
# Get Pod IP
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}'

# View network interfaces inside a pod
kubectl exec <pod-name> -- ip addr
kubectl exec <pod-name> -- ip route

# Check DNS configuration in pod
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

### 1.4 CNI (Container Network Interface)

CNI plugins configure Pod networking. Common plugins:

| Plugin | Description |
|--------|-------------|
| **Flannel** | Simple overlay network using VXLAN |
| **Calico** | L3 networking with BGP, Network Policies |
| **Weave** | Mesh network with encryption support |
| **Cilium** | eBPF-based with advanced security features |

```bash
# Check CNI configuration
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist

# Check CNI plugin pods
kubectl get pods -n kube-system -l k8s-app=calico-node  # Calico
kubectl get pods -n kube-system -l app=flannel          # Flannel
```

---

## 2. Define and Enforce Network Policies

### 2.1 Network Policy Basics

Network Policies control traffic flow to and from Pods at the IP/port level (Layer 3/4).

**Key Points:**
- Network Policies are **namespaced**
- By default, Pods are **non-isolated** (accept all traffic)
- Once a NetworkPolicy selects a Pod, that Pod is **isolated** for the policy types specified
- Network Policies are **additive** (allow rules, not deny)
- Requires a CNI plugin that supports Network Policies (Calico, Cilium, Weave)

### 2.2 Network Policy Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default
spec:
  podSelector:           # Which pods this policy applies to
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:               # Incoming traffic rules
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    - namespaceSelector:
        matchLabels:
          environment: prod
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    ports:
    - protocol: TCP
      port: 80
  egress:                # Outgoing traffic rules
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
```

### 2.3 Selectors

#### Pod Selector

```yaml
# Selects pods with label app=web
podSelector:
  matchLabels:
    app: web
```

#### Namespace Selector

```yaml
# Allows from all pods in namespaces with label env=prod
namespaceSelector:
  matchLabels:
    env: prod
```

#### Combined Pod and Namespace Selector

```yaml
# Pods with app=frontend in namespaces with env=prod
- namespaceSelector:
    matchLabels:
      env: prod
  podSelector:
    matchLabels:
      app: frontend
```

**Important:** The placement of `-` matters!

```yaml
# AND condition (pods matching BOTH selectors)
- namespaceSelector:
    matchLabels:
      env: prod
  podSelector:
    matchLabels:
      app: frontend

# OR condition (pods matching EITHER selector)
- namespaceSelector:
    matchLabels:
      env: prod
- podSelector:
    matchLabels:
      app: frontend
```

### 2.4 Common Network Policy Patterns

#### Default Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

#### Default Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

#### Default Deny All (Ingress and Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### Allow All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}
```

#### Allow from Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 5432
```

#### Allow DNS Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### 2.5 Network Policy Commands

```bash
# List network policies
kubectl get networkpolicies
kubectl get netpol -A

# Describe policy
kubectl describe networkpolicy <policy-name>

# Test connectivity (may require temporarily creating test pods)
kubectl run test-pod --image=busybox --rm -it -- wget -qO- --timeout=2 <target-ip>:<port>
```

---

## 3. Use ClusterIP, NodePort, LoadBalancer Service Types and Endpoints

### 3.1 Service Overview

A Service is an abstraction that defines a logical set of Pods and a policy to access them.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80           # Service port
    targetPort: 8080   # Pod port
```

### 3.2 Service Types

#### ClusterIP (Default)

Exposes the Service on a cluster-internal IP. Only accessible from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Create ClusterIP service imperatively
kubectl expose deployment myapp --port=80 --target-port=8080

# Access from within cluster
kubectl run test --image=busybox --rm -it -- wget -qO- clusterip-service:80
```

#### NodePort

Exposes the Service on each Node's IP at a static port (30000-32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080    # Optional: auto-assigned if not specified
```

```bash
# Create NodePort service
kubectl expose deployment myapp --port=80 --target-port=8080 --type=NodePort

# Access from outside cluster
curl <node-ip>:30080
```

#### LoadBalancer

Exposes the Service using a cloud provider's load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Create LoadBalancer service
kubectl expose deployment myapp --port=80 --target-port=8080 --type=LoadBalancer

# Get external IP
kubectl get svc loadbalancer-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

#### ExternalName

Maps the Service to a DNS name (no proxying).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.external.com
```

### 3.3 Headless Services

Services without a cluster IP, used for stateful applications.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None      # Makes it headless
  selector:
    app: database
  ports:
  - port: 5432
```

### 3.4 Endpoints

Endpoints are automatically created for Services with selectors.

```bash
# View endpoints
kubectl get endpoints
kubectl get ep <service-name>
kubectl describe ep <service-name>
```

#### Manual Endpoints (for external services)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 5432
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db    # Must match Service name
subsets:
  - addresses:
    - ip: 192.168.1.100
    - ip: 192.168.1.101
    ports:
    - port: 5432
```

### 3.5 Service Commands

```bash
# Create service
kubectl create service clusterip my-svc --tcp=80:8080
kubectl create service nodeport my-svc --tcp=80:8080 --node-port=30080
kubectl expose deployment nginx --port=80 --type=NodePort

# Get services
kubectl get svc
kubectl get svc -o wide
kubectl describe svc <service-name>

# Get service YAML
kubectl get svc <service-name> -o yaml
```

---

## 4. Use the Gateway API to Manage Ingress Traffic

### 4.1 Gateway API Overview

Gateway API is the successor to Ingress, providing more expressive and extensible traffic routing.

**Key Resources:**
- **GatewayClass**: Defines gateway implementation (like IngressClass)
- **Gateway**: Infrastructure configuration (load balancer)
- **HTTPRoute**: HTTP routing rules
- **GRPCRoute**: gRPC routing rules

### 4.2 GatewayClass

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-gateway-class
spec:
  controllerName: example.com/gateway-controller
```

### 4.3 Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-gateway-class
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: "*.example.com"
    allowedRoutes:
      namespaces:
        from: All
```

### 4.4 HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-route
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
```

### 4.5 Install Gateway API

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

---

## 5. Know How to Use Ingress Controllers and Ingress Resources

### 5.1 Ingress Overview

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster.

**Note:** Ingress API is frozen; Gateway API is the recommended future approach.

### 5.2 Ingress Controllers

An Ingress controller is required to fulfill Ingress resources.

Common Ingress Controllers:
- **NGINX Ingress Controller**
- **Traefik**
- **HAProxy**
- **Kong**
- **Contour**

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml

# Check installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### 5.3 Ingress Resource

#### Simple Fanout

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

#### Name-Based Virtual Hosting

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

#### TLS Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: tls-secret    # TLS secret with tls.crt and tls.key
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 443
```

```bash
# Create TLS secret
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

### 5.4 Path Types

| PathType | Description |
|----------|-------------|
| `Prefix` | Matches based on URL path prefix |
| `Exact` | Matches exact URL path |
| `ImplementationSpecific` | Interpretation depends on IngressClass |

### 5.5 Ingress Commands

```bash
# Create ingress imperatively
kubectl create ingress my-ingress --rule="host/path=service:port"

# Example
kubectl create ingress example --rule="example.com/api=api-svc:80" --rule="example.com/web=web-svc:80"

# Get ingress
kubectl get ingress
kubectl describe ingress <name>

# Get IngressClasses
kubectl get ingressclass
```

---

## 6. Understand and Use CoreDNS

### 6.1 CoreDNS Overview

CoreDNS is the default DNS server in Kubernetes, providing service discovery and name resolution.

### 6.2 DNS Records Created by Kubernetes

| Record Type | Format | Example |
|-------------|--------|---------|
| Service A/AAAA | `<service>.<namespace>.svc.cluster.local` | `nginx.default.svc.cluster.local` |
| Pod A/AAAA | `<pod-ip-dashed>.<namespace>.pod.cluster.local` | `10-244-0-5.default.pod.cluster.local` |
| SRV | `_<port>._<protocol>.<service>.<namespace>.svc.cluster.local` | `_http._tcp.nginx.default.svc.cluster.local` |

### 6.3 CoreDNS Configuration

```bash
# View CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### 6.4 Customizing CoreDNS

#### Add Custom DNS Entries

```yaml
# Add to Corefile
hosts custom.hosts example.com {
    192.168.1.100 custom.example.com
    fallthrough
}
```

#### Forward to Custom DNS Server

```yaml
# Add to Corefile
forward example.com 10.0.0.53
```

### 6.5 DNS Troubleshooting

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS resolution
kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default

# Detailed DNS test
kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- nslookup -type=srv _https._tcp.kubernetes.default

# Check pod DNS config
kubectl exec <pod-name> -- cat /etc/resolv.conf

# Expected resolv.conf:
# nameserver 10.96.0.10
# search <namespace>.svc.cluster.local svc.cluster.local cluster.local
# ndots:5
```

### 6.6 Custom DNS Settings for Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  containers:
  - name: app
    image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - ns1.svc.cluster.local
    - my.dns.search.suffix
    options:
    - name: ndots
      value: "2"
```

#### DNS Policies

| Policy | Description |
|--------|-------------|
| `Default` | Inherits DNS from node |
| `ClusterFirst` | Uses cluster DNS (default for pods) |
| `ClusterFirstWithHostNet` | For pods with hostNetwork: true |
| `None` | Allows custom DNS config |

---

## 7. Practice Exercises

### Exercise 1: Create Network Policy

Create a Network Policy that:
- Applies to pods with label `app=database`
- Only allows ingress from pods with label `app=backend`
- Only allows port 5432

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```

</details>

### Exercise 2: Create Services

Create a ClusterIP service named `backend-svc` and a NodePort service named `frontend-svc` for deployments with matching names.

<details>
<summary>Solution</summary>

```bash
# ClusterIP
kubectl expose deployment backend --name=backend-svc --port=80 --target-port=8080

# NodePort
kubectl expose deployment frontend --name=frontend-svc --port=80 --target-port=8080 --type=NodePort
```

</details>

### Exercise 3: Create Ingress

Create an Ingress that routes:
- `api.example.com` to `api-service:80`
- `web.example.com` to `web-service:80`

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

</details>

### Exercise 4: DNS Troubleshooting

A service named `myapp` in namespace `production` is not resolving. How do you troubleshoot?

<details>
<summary>Solution</summary>

```bash
# 1. Verify service exists
kubectl get svc myapp -n production

# 2. Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 3. Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# 4. Test DNS resolution
kubectl run dnstest --image=busybox:1.28 --rm -it -n production -- nslookup myapp

# 5. Try full DNS name
kubectl run dnstest --image=busybox:1.28 --rm -it -- nslookup myapp.production.svc.cluster.local

# 6. Check endpoints
kubectl get endpoints myapp -n production
```

</details>

---

## 8. Key Points for the Exam

1. **Pod Networking**: Every pod gets an IP, pods can communicate without NAT
2. **Network Policies**: Default allow all; once selected, pods are isolated
3. **Network Policy Selectors**: Understand AND vs OR conditions (hyphen placement)
4. **Service Types**: ClusterIP (internal), NodePort (30000-32767), LoadBalancer (cloud)
5. **Endpoints**: Created automatically for services with selectors
6. **Ingress**: Requires an Ingress controller; path types matter
7. **Gateway API**: Modern replacement for Ingress (know basics)
8. **CoreDNS**: Service discovery format `<service>.<namespace>.svc.cluster.local`

---

## 9. Quick Reference

### Service DNS Names

```
<service>.<namespace>.svc.cluster.local
<service>.<namespace>.svc
<service>.<namespace>
<service>  # Same namespace only
```

### Useful Commands

```bash
# Test service connectivity
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- <service>:<port>

# Test DNS
kubectl run test --image=busybox:1.28 --rm -it -- nslookup <service>

# Check network policy effect
kubectl get netpol
kubectl describe netpol <policy-name>

# Debug CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns -f
```

---

## 10. Documentation References

- https://kubernetes.io/docs/concepts/services-networking/
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://gateway-api.sigs.k8s.io/