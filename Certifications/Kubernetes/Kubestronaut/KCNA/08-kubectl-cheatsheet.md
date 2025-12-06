# kubectl Command Cheat Sheet for KCNA

## Quick Reference - Know These by Heart!

---

## Cluster Information

```bash
# Display cluster info
kubectl cluster-info

# View cluster configuration
kubectl config view

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Set default namespace
kubectl config set-context --current --namespace=<namespace>

# View API resources available
kubectl api-resources

# View API versions
kubectl api-versions

# Get cluster component status
kubectl get componentstatuses  # or: kubectl get cs
```

---

## Viewing Resources (GET)

```bash
# Basic get commands
kubectl get nodes                    # List nodes
kubectl get pods                     # List pods (current namespace)
kubectl get pods -A                  # List pods (all namespaces)
kubectl get pods -n <namespace>      # List pods (specific namespace)
kubectl get deployments              # List deployments
kubectl get services                 # List services
kubectl get all                      # List all resources

# Output formats
kubectl get pods -o wide             # More details (IP, node)
kubectl get pods -o yaml             # YAML output
kubectl get pods -o json             # JSON output
kubectl get pods -o name             # Just names

# Filtering
kubectl get pods -l app=nginx        # Filter by label
kubectl get pods --field-selector status.phase=Running
kubectl get pods --sort-by=.metadata.name

# Watch for changes
kubectl get pods -w                  # Watch mode
```

---

## Detailed Information (DESCRIBE)

```bash
kubectl describe node <node-name>
kubectl describe pod <pod-name>
kubectl describe deployment <name>
kubectl describe service <name>
kubectl describe configmap <name>
kubectl describe secret <name>
```

---

## Creating Resources

### Imperative Commands (Quick)
```bash
# Create Pod
kubectl run nginx --image=nginx

# Create Deployment
kubectl create deployment nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=3

# Create Service
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Create ConfigMap
kubectl create configmap my-config --from-literal=key=value
kubectl create configmap my-config --from-file=config.txt

# Create Secret
kubectl create secret generic my-secret --from-literal=password=secret
kubectl create secret generic my-secret --from-file=ssh-key=~/.ssh/id_rsa

# Create Namespace
kubectl create namespace <name>

# Create Job
kubectl create job my-job --image=busybox -- echo "Hello"

# Create CronJob
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo "Hello"
```

### Declarative (From YAML)
```bash
kubectl apply -f manifest.yaml       # Create or update
kubectl create -f manifest.yaml      # Create only
kubectl replace -f manifest.yaml     # Update only
kubectl delete -f manifest.yaml      # Delete
```

### Generate YAML (Don't memorize YAML!)
```bash
# Generate Pod YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Generate Deployment YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml

# Generate Service YAML
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml > service.yaml
```

---

## Editing Resources

```bash
# Edit resource directly
kubectl edit deployment nginx
kubectl edit pod nginx

# Patch resource
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'

# Set image
kubectl set image deployment/nginx nginx=nginx:1.21

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Autoscale
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

---

## Deleting Resources

```bash
kubectl delete pod nginx
kubectl delete deployment nginx
kubectl delete service nginx
kubectl delete -f manifest.yaml

# Force delete
kubectl delete pod nginx --force --grace-period=0

# Delete all pods in namespace
kubectl delete pods --all -n <namespace>

# Delete all resources with label
kubectl delete all -l app=nginx
```

---

## Logs and Debugging

```bash
# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>    # Multi-container pod
kubectl logs <pod-name> --previous        # Previous instance
kubectl logs <pod-name> -f                # Stream logs
kubectl logs -l app=nginx                 # By label

# Execute command in pod
kubectl exec <pod-name> -- ls /
kubectl exec -it <pod-name> -- /bin/bash  # Interactive shell
kubectl exec -it <pod-name> -c <container> -- /bin/sh

# Copy files
kubectl cp <pod-name>:/path/to/file ./local-file
kubectl cp ./local-file <pod-name>:/path/to/file

# Port forwarding
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80

# Debug with ephemeral container
kubectl debug <pod-name> -it --image=busybox

# Get events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -w
```

---

## Resource Management

```bash
# View resource usage (requires metrics-server)
kubectl top nodes
kubectl top pods
kubectl top pods --sort-by=memory

# Describe resource quotas
kubectl describe resourcequota -n <namespace>

# View limits
kubectl describe limitrange -n <namespace>
```

---

## Rollout Management

```bash
# Check rollout status
kubectl rollout status deployment/nginx

# View rollout history
kubectl rollout history deployment/nginx

# Rollback
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2

# Pause/Resume rollout
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx

# Restart (rolling restart)
kubectl rollout restart deployment/nginx
```

---

## Labels and Annotations

```bash
# Add label
kubectl label pod nginx env=prod

# Remove label
kubectl label pod nginx env-

# Overwrite label
kubectl label pod nginx env=dev --overwrite

# Add annotation
kubectl annotate pod nginx description="My pod"

# Remove annotation
kubectl annotate pod nginx description-
```

---

## Taints and Tolerations

```bash
# Taint a node
kubectl taint nodes node1 key=value:NoSchedule

# Remove taint
kubectl taint nodes node1 key=value:NoSchedule-

# View node taints
kubectl describe node node1 | grep Taints
```

---

## Node Management

```bash
# Mark node unschedulable
kubectl cordon node1

# Mark node schedulable
kubectl uncordon node1

# Drain node (evict pods)
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data
```

---

## Context and Namespace Shortcuts

```bash
# Aliases (add to .bashrc)
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'

# kubectl autocomplete (bash)
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Make 'k' work with autocomplete
complete -o default -F __start_kubectl k
```

---

## Kustomize with kubectl

```bash
# Preview kustomize output
kubectl kustomize <directory>

# Apply kustomize
kubectl apply -k <directory>

# Delete kustomize resources
kubectl delete -k <directory>
```

---

## Quick Reference Table

| Action | Command |
|--------|---------|
| List pods | `kubectl get pods` |
| All namespaces | `kubectl get pods -A` |
| Wide output | `kubectl get pods -o wide` |
| YAML output | `kubectl get pod nginx -o yaml` |
| Describe | `kubectl describe pod nginx` |
| Logs | `kubectl logs nginx` |
| Stream logs | `kubectl logs nginx -f` |
| Shell access | `kubectl exec -it nginx -- /bin/bash` |
| Create deployment | `kubectl create deployment nginx --image=nginx` |
| Scale | `kubectl scale deployment nginx --replicas=3` |
| Expose | `kubectl expose deployment nginx --port=80` |
| Delete | `kubectl delete pod nginx` |
| Apply YAML | `kubectl apply -f file.yaml` |
| Generate YAML | `kubectl create deployment nginx --image=nginx --dry-run=client -o yaml` |

---

## KCNA Exam Tips for kubectl

1. **You don't need deep kubectl mastery for KCNA** - it's conceptual
2. **Know what commands DO** - not necessarily exact syntax
3. **Understand the output** of `kubectl get` and `kubectl describe`
4. **Know imperative vs declarative** approaches
5. **Practice on a real cluster** to build familiarity

---

## Common Exam Questions About kubectl

Q: How do you list pods in all namespaces?
A: `kubectl get pods -A` or `kubectl get pods --all-namespaces`

Q: How do you get YAML definition of a running pod?
A: `kubectl get pod <n> -o yaml`

Q: How do you create a deployment imperatively?
A: `kubectl create deployment <n> --image=<image>`

Q: How do you check deployment rollout status?
A: `kubectl rollout status deployment/<n>`

Q: How do you rollback a deployment?
A: `kubectl rollout undo deployment/<n>`