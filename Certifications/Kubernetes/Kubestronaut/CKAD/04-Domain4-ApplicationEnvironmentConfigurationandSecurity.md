# CKAD Domain 4: Application Environment, Configuration and Security (25%)

## Overview

This is the largest domain in the CKAD exam (25%). It covers managing application configurations, securing workloads, controlling access, and extending Kubernetes functionality. You must understand ConfigMaps, Secrets, ServiceAccounts, SecurityContexts, RBAC, resource management, and Custom Resource Definitions (CRDs).

---

## 1. Discover and Use Resources that Extend Kubernetes (CRDs)

### What are Custom Resource Definitions?

CRDs extend Kubernetes by adding new resource types. They allow you to create your own API objects that work like native Kubernetes resources.

### Viewing Available CRDs

```bash
# List all CRDs in the cluster
kubectl get crd
kubectl get customresourcedefinitions

# Describe a specific CRD
kubectl describe crd certificates.cert-manager.io

# Get details of a CRD
kubectl get crd widgets.example.com -o yaml
```

### CRD Structure

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  group: example.com
  names:
    kind: Widget
    listKind: WidgetList
    plural: widgets
    singular: widget
    shortNames:
    - wg
  scope: Namespaced  # or Cluster
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string
              color:
                type: string
            required:
            - size
```

### Creating and Using Custom Resources

**Create the CRD:**
```bash
kubectl apply -f widget-crd.yaml
```

**Create a Custom Resource:**
```yaml
apiVersion: example.com/v1
kind: Widget
metadata:
  name: my-widget
spec:
  size: large
  color: blue
```

**Manage Custom Resources:**
```bash
# Create custom resource
kubectl apply -f my-widget.yaml

# List custom resources
kubectl get widgets
kubectl get wg  # Using short name

# Describe custom resource
kubectl describe widget my-widget

# Delete custom resource
kubectl delete widget my-widget
```

### Understanding Operators

Operators are controllers that use CRDs to automate complex application management. Common operators include:
- Prometheus Operator
- cert-manager
- Database operators (MySQL, PostgreSQL)

```bash
# Check if an operator is running
kubectl get pods -n operators

# View operator logs
kubectl logs -n operators deployment/prometheus-operator
```

---

## 2. Understand Authentication, Authorization, and Admission Control

### Authentication

Authentication verifies WHO you are. Kubernetes supports multiple authentication methods:

- **X.509 Client Certificates**
- **Bearer Tokens** (ServiceAccount tokens)
- **OpenID Connect (OIDC)**
- **Webhook Token Authentication**

**ServiceAccount Tokens:**
```bash
# View ServiceAccount token
kubectl get secret $(kubectl get sa default -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 -d
```

### Authorization

Authorization determines WHAT you can do. Kubernetes supports:

- **RBAC** (Role-Based Access Control) - Most common
- Node Authorization
- Webhook Authorization
- ABAC (Attribute-Based Access Control)

**Check your permissions:**
```bash
# Can I create pods?
kubectl auth can-i create pods

# Can I delete deployments in kube-system?
kubectl auth can-i delete deployments -n kube-system

# Check as another user
kubectl auth can-i create pods --as=system:serviceaccount:default:mysa

# List all permissions
kubectl auth can-i --list
kubectl auth can-i --list --as=system:serviceaccount:myns:mysa
```

### Admission Control

Admission controllers intercept requests AFTER authentication/authorization but BEFORE persistence. They can:
- **Mutate** - Modify the request
- **Validate** - Accept or reject the request

Common admission controllers:
- **NamespaceLifecycle** - Prevents operations in non-existent namespaces
- **LimitRanger** - Enforces default limits
- **ResourceQuota** - Enforces quotas
- **PodSecurity** - Enforces Pod Security Standards

---

## 3. Understanding RBAC

RBAC uses Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings to control access.

### Role (Namespaced)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]  # "" indicates core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### ClusterRole (Cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### RoleBinding (Namespaced)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: myapp-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding (Cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: managers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Common RBAC Verbs

| Verb | Description |
|------|-------------|
| get | Read a single resource |
| list | List all resources of a type |
| watch | Watch for changes |
| create | Create new resources |
| update | Update existing resources |
| patch | Partially update resources |
| delete | Delete resources |
| deletecollection | Delete multiple resources |

### Imperative RBAC Commands

```bash
# Create a Role
kubectl create role pod-reader --verb=get,list,watch --resource=pods

# Create a ClusterRole
kubectl create clusterrole secret-reader --verb=get,list,watch --resource=secrets

# Create RoleBinding
kubectl create rolebinding read-pods --role=pod-reader --user=jane --serviceaccount=default:mysa

# Create ClusterRoleBinding
kubectl create clusterrolebinding read-secrets --clusterrole=secret-reader --user=admin

# Check roles and bindings
kubectl get roles,rolebindings
kubectl get clusterroles,clusterrolebindings
kubectl describe role pod-reader
kubectl describe rolebinding read-pods
```

---

## 4. Understanding Resource Requirements, Limits, and Quotas

### Resource Requests and Limits

**Requests**: Minimum resources guaranteed to the container
**Limits**: Maximum resources the container can use

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### CPU Units

- `1` = 1 vCPU/Core
- `1000m` = 1 vCPU (m = millicpu)
- `500m` = 0.5 vCPU
- `100m` = 0.1 vCPU

### Memory Units

- `128Mi` = 128 Mebibytes
- `1Gi` = 1 Gibibyte
- `128M` = 128 Megabytes (decimal)

### LimitRange (Namespace Defaults)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limit-range
  namespace: default
spec:
  limits:
  - default:  # Default limits
      cpu: 500m
      memory: 512Mi
    defaultRequest:  # Default requests
      cpu: 100m
      memory: 128Mi
    max:  # Maximum allowed
      cpu: "2"
      memory: 2Gi
    min:  # Minimum required
      cpu: 50m
      memory: 64Mi
    type: Container
```

### ResourceQuota (Namespace Limits)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: default
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "10"
    persistentvolumeclaims: "5"
    configmaps: "10"
    secrets: "10"
    services: "5"
```

### Managing Quotas

```bash
# View quotas
kubectl get resourcequota
kubectl describe resourcequota compute-quota

# View limit ranges
kubectl get limitrange
kubectl describe limitrange cpu-mem-limit-range

# Check namespace usage against quota
kubectl describe namespace default | grep -A 20 "Resource Quotas"
```

---

## 5. Understand ConfigMaps

ConfigMaps store non-confidential configuration data as key-value pairs.

### Creating ConfigMaps

**Imperative from literals:**
```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=DATABASE_URL=postgres://localhost:5432/db
```

**Imperative from file:**
```bash
# Create from file (key = filename)
kubectl create configmap app-config --from-file=config.properties

# Create from file with custom key
kubectl create configmap app-config --from-file=app.conf=config.properties

# Create from directory
kubectl create configmap app-config --from-file=config-dir/
```

**Declarative:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_DEBUG: "false"
  DATABASE_URL: postgres://localhost:5432/db
  config.json: |
    {
      "logLevel": "info",
      "maxRetries": 3
    }
```

### Using ConfigMaps in Pods

**As Environment Variables:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    # Single key from ConfigMap
    - name: APP_ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    # All keys from ConfigMap as env vars
    envFrom:
    - configMapRef:
        name: app-config
        prefix: CONFIG_  # Optional prefix
```

**As Volume:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      # Optional: specify which keys to include
      items:
      - key: config.json
        path: app-config.json
      defaultMode: 0644
```

### Immutable ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
immutable: true  # Cannot be changed after creation
```

---

## 6. Create and Consume Secrets

Secrets store sensitive data like passwords, tokens, and keys.

### Creating Secrets

**Imperative:**
```bash
# Generic secret from literals
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cr3t!'

# From files
kubectl create secret generic tls-certs \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=admin \
  --docker-password=secret \
  --docker-email=admin@example.com

# TLS secret
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

**Declarative (values must be base64 encoded):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=      # echo -n 'admin' | base64
  password: UzNjcjN0IQ==  # echo -n 'S3cr3t!' | base64
```

**Using stringData (automatically encoded):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: S3cr3t!
```

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Generic secret (default) |
| `kubernetes.io/service-account-token` | ServiceAccount token |
| `kubernetes.io/dockerconfigjson` | Docker registry auth |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/basic-auth` | Basic authentication |

### Using Secrets in Pods

**As Environment Variables:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    # Single key from Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    # All keys from Secret
    envFrom:
    - secretRef:
        name: db-credentials
        prefix: DB_
```

**As Volume:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400  # Read-only for owner
      items:
      - key: password
        path: db-password
```

### ImagePullSecrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: registry.example.com/myapp:v1
  imagePullSecrets:
  - name: regcred
```

### Managing Secrets

```bash
# View secrets
kubectl get secrets
kubectl describe secret db-credentials

# Decode a secret value
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d

# Edit a secret
kubectl edit secret db-credentials
```

---

## 7. Understand ServiceAccounts

ServiceAccounts provide identities for pods to interact with the Kubernetes API.

### Default ServiceAccount

Every namespace has a `default` ServiceAccount that pods use if none is specified.

```bash
# List ServiceAccounts
kubectl get serviceaccounts
kubectl get sa

# Describe default ServiceAccount
kubectl describe sa default
```

### Creating ServiceAccounts

**Imperative:**
```bash
kubectl create serviceaccount myapp-sa
```

**Declarative:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
automountServiceAccountToken: false  # Optional: disable token mounting
```

### Using ServiceAccounts in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: true  # Mount API token
  containers:
  - name: app
    image: myapp:v1
```

### ServiceAccount with ImagePullSecrets

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
imagePullSecrets:
- name: regcred
```

### ServiceAccount RBAC

```yaml
# Grant permissions to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-pods-access
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 8. Understand Application Security (SecurityContexts)

SecurityContexts define privilege and access control settings for pods and containers.

### Pod-Level SecurityContext

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    supplementalGroups: [4000]
  containers:
  - name: app
    image: myapp:v1
```

### Container-Level SecurityContext

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      runAsUser: 1000
      runAsNonRoot: true
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      privileged: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
```

### SecurityContext Fields

| Field | Description |
|-------|-------------|
| `runAsUser` | UID to run the container process |
| `runAsGroup` | GID to run the container process |
| `fsGroup` | GID for volume ownership |
| `runAsNonRoot` | Prevent running as root (UID 0) |
| `readOnlyRootFilesystem` | Mount root filesystem read-only |
| `allowPrivilegeEscalation` | Prevent gaining more privileges than parent |
| `privileged` | Run container in privileged mode |
| `capabilities` | Linux capabilities to add/drop |

### Complete Secure Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: data
      mountPath: /data
  volumes:
  - name: tmp
    emptyDir: {}
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

---

## 9. Essential kubectl Commands for This Domain

```bash
# ConfigMaps
kubectl create configmap myconfig --from-literal=key=value
kubectl create configmap myconfig --from-file=config.properties
kubectl get configmap myconfig -o yaml
kubectl describe configmap myconfig

# Secrets
kubectl create secret generic mysecret --from-literal=password=secret
kubectl create secret docker-registry regcred --docker-server=... --docker-username=...
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 -d

# ServiceAccounts
kubectl create serviceaccount mysa
kubectl get sa
kubectl describe sa mysa

# RBAC
kubectl create role pod-reader --verb=get,list --resource=pods
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:mysa
kubectl auth can-i create pods --as=system:serviceaccount:default:mysa
kubectl get roles,rolebindings

# Resource Quotas
kubectl create quota myquota --hard=pods=10,cpu=4,memory=4Gi
kubectl get resourcequota
kubectl describe resourcequota myquota

# CRDs
kubectl get crd
kubectl describe crd <crd-name>
kubectl get <custom-resource>
```

---

## Exam Tips for This Domain

1. **Master ConfigMaps and Secrets**: Know creation methods and mounting as env vars and volumes
2. **Understand RBAC**: Role vs ClusterRole, RoleBinding vs ClusterRoleBinding
3. **Know SecurityContext fields**: runAsUser, runAsNonRoot, readOnlyRootFilesystem, capabilities
4. **Practice ServiceAccounts**: Creation, RBAC binding, pod assignment
5. **Remember resource units**: CPU (m = millicpu), Memory (Mi, Gi)
6. **Know the difference**: Requests vs Limits, LimitRange vs ResourceQuota
7. **Base64 encoding**: Secrets require base64 encoded values (or use stringData)
8. **Use `kubectl auth can-i`**: Quick way to verify permissions

---

## Practice Exercises

1. Create a ConfigMap from a file and mount it as a volume in a pod
2. Create a Secret with username and password, inject as environment variables
3. Create a ServiceAccount, grant it permission to read pods, use it in a deployment
4. Create a Pod that runs as non-root user with read-only filesystem
5. Create a ResourceQuota limiting namespace to 5 pods and 2 CPU
6. Create a Role allowing get, list, watch on deployments, bind it to a ServiceAccount
7. Create a pod using a private image with imagePullSecrets
8. Create a LimitRange setting default CPU requests/limits for containers
9. Verify ServiceAccount permissions using `kubectl auth can-i --as`
10. Create a CRD and a custom resource instance