# CKAD Domain 5: Services and Networking (20%)

## Overview

This domain covers exposing applications to internal and external traffic using Services, Ingress, and NetworkPolicies. You must understand different Service types, how to configure Ingress for HTTP/HTTPS routing, and how to implement network security using NetworkPolicies.

---

## 1. Demonstrate Understanding of NetworkPolicies

NetworkPolicies control traffic flow to and from pods. By default, pods accept traffic from any source. NetworkPolicies implement a firewall at the pod level.

### NetworkPolicy Basics

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend  # Apply to pods with this label
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Policy Types

- **Ingress**: Controls incoming traffic to selected pods
- **Egress**: Controls outgoing traffic from selected pods
- If no `policyTypes` specified, defaults to Ingress (and Egress if egress rules exist)

### Selector Types

**podSelector**: Select pods in the same namespace
```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        role: frontend
```

**namespaceSelector**: Select pods in specific namespaces
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
```

**Combined podSelector AND namespaceSelector** (must match both):
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
    podSelector:
      matchLabels:
        role: frontend
```

**podSelector OR namespaceSelector** (match either):
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
  - podSelector:
      matchLabels:
        role: frontend
```

**ipBlock**: Allow traffic from specific IP ranges
```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.0.1.0/24
```

### Default Policies

**Default Deny All Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}  # Select all pods in namespace
  policyTypes:
  - Ingress
```

**Default Deny All Egress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

**Default Deny All Traffic:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Allow All Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
  - {}  # Empty rule allows all
  policyTypes:
  - Ingress
```

**Allow All Egress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

### Complete NetworkPolicy Examples

**Allow traffic only from specific pods:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
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
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Allow DNS egress (required for name resolution):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

**Multi-tier application policy:**
```yaml
# Allow frontend to access backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080
      protocol: TCP
---
# Allow backend to access database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 5432
      protocol: TCP
```

### Managing NetworkPolicies

```bash
# Create NetworkPolicy
kubectl apply -f networkpolicy.yaml

# List NetworkPolicies
kubectl get networkpolicies
kubectl get netpol

# Describe NetworkPolicy
kubectl describe networkpolicy allow-frontend

# Delete NetworkPolicy
kubectl delete networkpolicy allow-frontend

# Test connectivity
kubectl exec -it frontend-pod -- curl backend-service:8080
```

---

## 2. Provide and Troubleshoot Access to Applications via Services

### Service Types

| Type | Description | Use Case |
|------|-------------|----------|
| `ClusterIP` | Internal cluster IP (default) | Internal communication |
| `NodePort` | Exposes on each node's IP at static port | Development, direct access |
| `LoadBalancer` | External load balancer | Production cloud deployments |
| `ExternalName` | Maps to external DNS name | External service integration |

### ClusterIP Service (Default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # Default, can be omitted
  selector:
    app: backend
  ports:
  - name: http
    port: 80           # Service port
    targetPort: 8080   # Container port
    protocol: TCP
```

**Imperative:**
```bash
kubectl expose deployment backend --port=80 --target-port=8080
```

### NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Optional: 30000-32767, auto-assigned if omitted
```

**Imperative:**
```bash
kubectl expose deployment frontend --type=NodePort --port=80 --target-port=8080
```

**Access:**
```bash
# Access via any node IP
curl http://<node-ip>:30080
```

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Imperative:**
```bash
kubectl expose deployment web --type=LoadBalancer --port=80 --target-port=8080
```

**Check external IP:**
```bash
kubectl get service web-service
# EXTERNAL-IP column shows the load balancer IP
```

### ExternalName Service

Maps a service to an external DNS name (no selector needed):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

### Headless Service

For StatefulSets and when you need direct pod DNS:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None  # Makes it headless
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

### Multi-Port Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

### Service DNS

Services get DNS names in the format:
```
<service-name>.<namespace>.svc.cluster.local
```

**Examples:**
- `backend-service` (same namespace)
- `backend-service.default` (explicit namespace)
- `backend-service.default.svc.cluster.local` (FQDN)

### Troubleshooting Services

**1. Check Service exists and has correct selector:**
```bash
kubectl get service myservice -o yaml
kubectl describe service myservice
```

**2. Check Endpoints exist:**
```bash
kubectl get endpoints myservice
# Should list pod IPs if selector matches
```

**3. Verify pods match selector:**
```bash
# Get service selector
kubectl get service myservice -o jsonpath='{.spec.selector}'

# Check pods with matching labels
kubectl get pods -l app=myapp
```

**4. Test connectivity:**
```bash
# From another pod
kubectl run test --rm -it --image=busybox -- wget -qO- myservice:80

# DNS resolution
kubectl run test --rm -it --image=busybox -- nslookup myservice

# Check if port is open
kubectl run test --rm -it --image=busybox -- nc -zv myservice 80
```

**5. Check pod readiness:**
```bash
kubectl get pods -l app=myapp
# Pods must be Ready for traffic
```

**Common Issues:**

| Problem | Cause | Solution |
|---------|-------|----------|
| No endpoints | Selector doesn't match pod labels | Fix labels or selector |
| Connection refused | Pod not listening on targetPort | Verify container port |
| Service not found | Wrong namespace or name | Check namespace |
| Timeout | NetworkPolicy blocking traffic | Check NetworkPolicies |

---

## 3. Use Ingress Rules to Expose Applications

### Ingress Basics

Ingress exposes HTTP/HTTPS routes from outside the cluster to services within the cluster.

**Prerequisites:**
- Ingress controller must be installed (nginx, traefik, etc.)
- Ingress resources define the rules

### Simple Ingress (Single Service)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  defaultBackend:
    service:
      name: default-backend
      port:
        number: 80
```

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
  - http:
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

### Host-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
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

### Combined Host and Path Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Path Types

| Type | Description |
|------|-------------|
| `Prefix` | Matches based on URL path prefix (e.g., /api matches /api/users) |
| `Exact` | Matches the exact path only |
| `ImplementationSpecific` | Interpretation depends on IngressClass |

### TLS/HTTPS Configuration

**Create TLS Secret:**
```bash
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

**Ingress with TLS:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - secure.example.com
    secretName: tls-secret
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
              number: 80
```

### IngressClass

Specify which Ingress controller should handle the Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx  # Specify the ingress class
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

**View IngressClasses:**
```bash
kubectl get ingressclass
```

### Annotations

Common nginx ingress annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
spec:
  rules:
  - http:
      paths:
      - path: /app(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### Managing Ingress

```bash
# Create Ingress
kubectl apply -f ingress.yaml

# List Ingress resources
kubectl get ingress

# Describe Ingress
kubectl describe ingress my-ingress

# Get Ingress with more details
kubectl get ingress my-ingress -o yaml

# Edit Ingress
kubectl edit ingress my-ingress

# Delete Ingress
kubectl delete ingress my-ingress
```

### Troubleshooting Ingress

**1. Check Ingress resource:**
```bash
kubectl describe ingress my-ingress
# Look for Address and Backend status
```

**2. Check Ingress controller logs:**
```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

**3. Verify backend service:**
```bash
kubectl get service app-service
kubectl get endpoints app-service
```

**4. Test from inside cluster:**
```bash
kubectl run test --rm -it --image=busybox -- wget -qO- app-service:80
```

**5. Check DNS resolution:**
```bash
# Ensure hostname resolves to Ingress IP
nslookup app.example.com
```

---

## 4. Service Discovery and DNS

### Kubernetes DNS

Every Service gets a DNS entry automatically:
- `<service>.<namespace>.svc.cluster.local`

### DNS for Services

```bash
# Same namespace - use service name
curl http://backend-service

# Different namespace - include namespace
curl http://backend-service.other-namespace

# Full FQDN
curl http://backend-service.other-namespace.svc.cluster.local
```

### DNS for Pods

Pods can have DNS records (when enabled):
- `<pod-ip-dashed>.<namespace>.pod.cluster.local`

Example: `10-244-0-5.default.pod.cluster.local`

### Headless Service DNS

For headless services (clusterIP: None), DNS returns pod IPs:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
```

**DNS queries:**
- `mysql.default.svc.cluster.local` → Returns all pod IPs
- For StatefulSets: `mysql-0.mysql.default.svc.cluster.local` → Specific pod

### Pod DNS Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns
spec:
  dnsPolicy: "None"  # Override cluster DNS
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - my.dns.search.suffix
    options:
    - name: ndots
      value: "2"
  containers:
  - name: app
    image: myapp
```

**DNS Policies:**
- `ClusterFirst` (default): Use cluster DNS, fallback to upstream
- `Default`: Use node's DNS configuration
- `ClusterFirstWithHostNet`: For hostNetwork pods
- `None`: Use custom dnsConfig

---

## 5. Essential kubectl Commands for This Domain

```bash
# Services
kubectl expose deployment myapp --port=80 --target-port=8080
kubectl expose deployment myapp --type=NodePort --port=80
kubectl expose deployment myapp --type=LoadBalancer --port=80
kubectl get services
kubectl describe service myservice
kubectl get endpoints myservice

# Ingress
kubectl get ingress
kubectl describe ingress myingress
kubectl get ingressclass

# NetworkPolicies
kubectl get networkpolicies
kubectl describe networkpolicy mypolicy

# Troubleshooting
kubectl run test --rm -it --image=busybox -- wget -qO- myservice:80
kubectl run test --rm -it --image=busybox -- nslookup myservice
kubectl exec -it mypod -- curl http://otherservice:8080

# Labels and Selectors
kubectl get pods -l app=myapp
kubectl get pods --show-labels
kubectl label pod mypod tier=frontend
```

---

## Exam Tips for This Domain

1. **Know all Service types**: ClusterIP, NodePort, LoadBalancer, ExternalName
2. **Understand Service DNS**: `<service>.<namespace>.svc.cluster.local`
3. **Master NetworkPolicy selectors**: podSelector, namespaceSelector, ipBlock
4. **Remember NetworkPolicy logic**: AND vs OR when combining selectors
5. **Know default deny patterns**: Empty podSelector `{}` selects all pods
6. **Understand Ingress path types**: Prefix vs Exact
7. **Practice troubleshooting**: Check endpoints, test connectivity, examine events
8. **Remember port terminology**: port (service), targetPort (container), nodePort

---

## Practice Exercises

1. Create a ClusterIP service for a deployment, verify endpoints are populated
2. Create a NodePort service and access it via node IP
3. Create a NetworkPolicy that allows ingress only from pods with label `tier=frontend`
4. Create a default-deny NetworkPolicy, then add rules to allow specific traffic
5. Create an Ingress with path-based routing: /api → api-service, /web → web-service
6. Create an Ingress with host-based routing for two different domains
7. Troubleshoot a service with no endpoints (fix label mismatch)
8. Create a NetworkPolicy allowing egress only to pods with label `app=database` on port 5432
9. Configure TLS for an Ingress using a Secret
10. Create a headless service for a StatefulSet

---

## Quick Reference: Common Service Patterns

### Internal Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-svc
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### External NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-svc
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

### Default Deny + Allow Pattern
```yaml
# Step 1: Deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Step 2: Allow specific
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

### Complete Ingress Example
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```