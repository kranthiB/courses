# CKA kubectl Speed Drills

## Purpose

These drills are designed to build **muscle memory** for kubectl commands. Practice each drill until you can complete it in the target time WITHOUT looking at the solutions.

**Golden Rule:** Repeat each drill at least 10 times over multiple days.

---

## Setup Before Practice

```bash
# Set up aliases (already available in exam, but practice using them)
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias kds='kubectl describe svc'

# Useful exports
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
```

---

## Drill Set 1: Pod Operations (Target: 30 seconds each)

### Drill 1.1: Create a simple pod
```bash
# Create pod named "nginx" with image nginx
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run nginx --image=nginx
```
</details>

### Drill 1.2: Create pod with specific port
```bash
# Create pod "web" with image nginx exposing port 80
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run web --image=nginx --port=80
```
</details>

### Drill 1.3: Create pod and generate YAML
```bash
# Generate YAML for pod "test" with image busybox (don't create)
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run test --image=busybox --dry-run=client -o yaml
# Or with alias:
k run test --image=busybox $do
```
</details>

### Drill 1.4: Create pod with command
```bash
# Create pod "sleeper" with busybox that sleeps for 3600 seconds
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run sleeper --image=busybox --command -- sleep 3600
```
</details>

### Drill 1.5: Create pod with labels
```bash
# Create pod "labeled" with nginx, labels: app=web, tier=frontend
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run labeled --image=nginx --labels="app=web,tier=frontend"
```
</details>

### Drill 1.6: Create pod in specific namespace
```bash
# Create pod "nginx" in namespace "production"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run nginx --image=nginx -n production
```
</details>

### Drill 1.7: Delete pod immediately
```bash
# Delete pod "nginx" immediately (force)
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k delete pod nginx --force --grace-period=0
# Or with alias:
k delete pod nginx $now
```
</details>

---

## Drill Set 2: Deployment Operations (Target: 45 seconds each)

### Drill 2.1: Create deployment
```bash
# Create deployment "webapp" with nginx image, 3 replicas
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create deployment webapp --image=nginx --replicas=3
```
</details>

### Drill 2.2: Scale deployment
```bash
# Scale deployment "webapp" to 5 replicas
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k scale deployment webapp --replicas=5
```
</details>

### Drill 2.3: Update deployment image
```bash
# Update deployment "webapp" container "nginx" to nginx:1.21
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k set image deployment/webapp nginx=nginx:1.21
```
</details>

### Drill 2.4: Rollback deployment
```bash
# Rollback deployment "webapp" to previous version
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k rollout undo deployment/webapp
```
</details>

### Drill 2.5: Check rollout status
```bash
# Check rollout status of deployment "webapp"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k rollout status deployment/webapp
```
</details>

### Drill 2.6: View rollout history
```bash
# View rollout history of deployment "webapp"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k rollout history deployment/webapp
```
</details>

### Drill 2.7: Rollback to specific revision
```bash
# Rollback deployment "webapp" to revision 2
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k rollout undo deployment/webapp --to-revision=2
```
</details>

---

## Drill Set 3: Service Operations (Target: 30 seconds each)

### Drill 3.1: Expose deployment as ClusterIP
```bash
# Expose deployment "webapp" on port 80
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k expose deployment webapp --port=80
```
</details>

### Drill 3.2: Expose as NodePort
```bash
# Expose deployment "webapp" as NodePort on port 80
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k expose deployment webapp --port=80 --type=NodePort
```
</details>

### Drill 3.3: Create service with target port
```bash
# Expose deployment "webapp" port 80, target port 8080
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k expose deployment webapp --port=80 --target-port=8080
```
</details>

### Drill 3.4: Create service for pod
```bash
# Expose pod "nginx" on port 80
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k expose pod nginx --port=80
```
</details>

### Drill 3.5: Check endpoints
```bash
# Get endpoints for service "webapp"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get endpoints webapp
# Or:
k get ep webapp
```
</details>

---

## Drill Set 4: ConfigMaps and Secrets (Target: 30 seconds each)

### Drill 4.1: Create ConfigMap from literals
```bash
# Create configmap "app-config" with KEY1=value1, KEY2=value2
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create configmap app-config --from-literal=KEY1=value1 --from-literal=KEY2=value2
```
</details>

### Drill 4.2: Create ConfigMap from file
```bash
# Create configmap "file-config" from file config.txt
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create configmap file-config --from-file=config.txt
```
</details>

### Drill 4.3: Create Secret from literals
```bash
# Create secret "db-creds" with user=admin, password=secret123
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create secret generic db-creds --from-literal=user=admin --from-literal=password=secret123
```
</details>

### Drill 4.4: Create TLS Secret
```bash
# Create TLS secret "tls-cert" from cert.crt and key.key
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create secret tls tls-cert --cert=cert.crt --key=key.key
```
</details>

### Drill 4.5: View secret value (decoded)
```bash
# Get decoded value of key "password" from secret "db-creds"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```
</details>

---

## Drill Set 5: RBAC (Target: 45 seconds each)

### Drill 5.1: Create ServiceAccount
```bash
# Create serviceaccount "my-sa" in namespace "dev"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create serviceaccount my-sa -n dev
```
</details>

### Drill 5.2: Create Role
```bash
# Create role "pod-reader" that can get, list, watch pods
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create role pod-reader --verb=get,list,watch --resource=pods
```
</details>

### Drill 5.3: Create RoleBinding
```bash
# Bind role "pod-reader" to serviceaccount "my-sa" in namespace "dev"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=dev:my-sa -n dev
```
</details>

### Drill 5.4: Create ClusterRole
```bash
# Create clusterrole "node-reader" that can get, list nodes
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create clusterrole node-reader --verb=get,list --resource=nodes
```
</details>

### Drill 5.5: Create ClusterRoleBinding
```bash
# Bind clusterrole "node-reader" to user "jane"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create clusterrolebinding node-reader-binding --clusterrole=node-reader --user=jane
```
</details>

### Drill 5.6: Check permissions
```bash
# Check if user "jane" can list pods in namespace "dev"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k auth can-i list pods --as=jane -n dev
```
</details>

### Drill 5.7: List all permissions for a user
```bash
# List all permissions for user "jane" in namespace "dev"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k auth can-i --list --as=jane -n dev
```
</details>

---

## Drill Set 6: Debugging and Logs (Target: 20 seconds each)

### Drill 6.1: View pod logs
```bash
# Get logs from pod "nginx"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k logs nginx
```
</details>

### Drill 6.2: View logs from specific container
```bash
# Get logs from container "app" in pod "multi"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k logs multi -c app
```
</details>

### Drill 6.3: Follow logs
```bash
# Stream logs from pod "nginx"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k logs nginx -f
```
</details>

### Drill 6.4: Previous container logs
```bash
# Get logs from previous container instance of pod "nginx"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k logs nginx --previous
```
</details>

### Drill 6.5: Exec into pod
```bash
# Open shell in pod "nginx"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k exec -it nginx -- /bin/sh
# Or for bash:
k exec -it nginx -- /bin/bash
```
</details>

### Drill 6.6: Run command in pod
```bash
# Run "cat /etc/hosts" in pod "nginx"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k exec nginx -- cat /etc/hosts
```
</details>

### Drill 6.7: Port forward
```bash
# Forward local port 8080 to pod "nginx" port 80
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k port-forward pod/nginx 8080:80
```
</details>

---

## Drill Set 7: Resource Information (Target: 15 seconds each)

### Drill 7.1: Get pods with wide output
```bash
# List pods with node and IP information
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get pods -o wide
```
</details>

### Drill 7.2: Get pods in all namespaces
```bash
# List pods in all namespaces
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get pods -A
# Or:
k get pods --all-namespaces
```
</details>

### Drill 7.3: Get pod IP using jsonpath
```bash
# Get IP of pod "nginx" using jsonpath
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get pod nginx -o jsonpath='{.status.podIP}'
```
</details>

### Drill 7.4: Get pods sorted by creation time
```bash
# List pods sorted by creation time
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get pods --sort-by=.metadata.creationTimestamp
```
</details>

### Drill 7.5: Get pods with labels
```bash
# Get pods showing labels
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get pods --show-labels
```
</details>

### Drill 7.6: Filter pods by label
```bash
# Get pods with label app=nginx
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k get pods -l app=nginx
```
</details>

### Drill 7.7: Get resource usage
```bash
# Show resource usage for all pods
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k top pods
```
</details>

---

## Drill Set 8: Node and Cluster Operations (Target: 30 seconds each)

### Drill 8.1: Cordon a node
```bash
# Prevent scheduling on node "worker-1"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k cordon worker-1
```
</details>

### Drill 8.2: Drain a node
```bash
# Drain node "worker-1" for maintenance
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k drain worker-1 --ignore-daemonsets --delete-emptydir-data
```
</details>

### Drill 8.3: Uncordon a node
```bash
# Allow scheduling on node "worker-1" again
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k uncordon worker-1
```
</details>

### Drill 8.4: Label a node
```bash
# Add label "disktype=ssd" to node "worker-1"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k label node worker-1 disktype=ssd
```
</details>

### Drill 8.5: Taint a node
```bash
# Taint node "worker-1" with key=value:NoSchedule
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k taint node worker-1 key=value:NoSchedule
```
</details>

### Drill 8.6: Remove taint
```bash
# Remove taint "key=value:NoSchedule" from node "worker-1"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k taint node worker-1 key=value:NoSchedule-
```
</details>

### Drill 8.7: Switch context
```bash
# Switch to context "cluster-2"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k config use-context cluster-2
```
</details>

---

## Drill Set 9: Network Testing (Target: 30 seconds each)

### Drill 9.1: Test service connectivity
```bash
# Test if service "nginx" is reachable on port 80
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run test --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- nginx:80
```
</details>

### Drill 9.2: Test DNS resolution
```bash
# Test DNS resolution for service "nginx"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx
```
</details>

### Drill 9.3: Full DNS name lookup
```bash
# Lookup full DNS name for service "nginx" in namespace "default"
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx.default.svc.cluster.local
```
</details>

---

## Drill Set 10: Quick YAML Generation (Target: 60 seconds each)

### Drill 10.1: Pod YAML with resources
```bash
# Generate YAML for pod with CPU/memory requests and limits
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k run nginx --image=nginx $do > pod.yaml
# Then edit to add:
# resources:
#   requests:
#     memory: "64Mi"
#     cpu: "250m"
#   limits:
#     memory: "128Mi"
#     cpu: "500m"
```
</details>

### Drill 10.2: Deployment YAML
```bash
# Generate deployment YAML with 3 replicas
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create deployment nginx --image=nginx --replicas=3 $do > deploy.yaml
```
</details>

### Drill 10.3: Service YAML
```bash
# Generate ClusterIP service YAML
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k expose deployment nginx --port=80 $do > svc.yaml
```
</details>

### Drill 10.4: Ingress YAML
```bash
# Generate ingress YAML
# Your command:

```
<details>
<summary>Solution</summary>

```bash
k create ingress myingress --rule="host/path=service:port" $do > ingress.yaml
```
</details>

---

## Speed Test: Comprehensive Drill

**Complete all tasks in 10 minutes:**

1. Create namespace "speed-test"
2. Create deployment "web" with nginx:1.19, 3 replicas
3. Expose deployment as ClusterIP service on port 80
4. Create configmap "app-config" with APP_ENV=production
5. Scale deployment to 5 replicas
6. Update image to nginx:1.20
7. Check rollout status
8. Create role "web-reader" that can get pods
9. Create rolebinding for serviceaccount "default"
10. Verify connectivity to service

<details>
<summary>Solution</summary>

```bash
k create ns speed-test
k create deployment web --image=nginx:1.19 --replicas=3 -n speed-test
k expose deployment web --port=80 -n speed-test
k create configmap app-config --from-literal=APP_ENV=production -n speed-test
k scale deployment web --replicas=5 -n speed-test
k set image deployment/web nginx=nginx:1.20 -n speed-test
k rollout status deployment/web -n speed-test
k create role web-reader --verb=get --resource=pods -n speed-test
k create rolebinding web-reader-binding --role=web-reader --serviceaccount=speed-test:default -n speed-test
k run test --image=busybox:1.28 --rm -it -n speed-test --restart=Never -- wget -qO- web
```
</details>

---

## Daily Practice Routine

| Time | Activity |
|------|----------|
| 5 min | Warm-up: Drill Sets 1-2 |
| 10 min | Focus: One drill set in depth |
| 5 min | Speed test |
| 5 min | Review mistakes |

**Weekly Goals:**
- Week 1: Master Drill Sets 1-4
- Week 2: Master Drill Sets 5-7
- Week 3: Master Drill Sets 8-10
- Week 4: Full speed tests daily