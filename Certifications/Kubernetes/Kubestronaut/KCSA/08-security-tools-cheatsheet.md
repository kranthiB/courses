# KCSA Security Tools Cheatsheet

## Quick Reference - Know These Tools!

---

## Vulnerability Scanning Tools

### Trivy (Aqua Security)
**Purpose**: Comprehensive vulnerability scanner for images, filesystems, and configs

```bash
# Scan container image
trivy image nginx:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan filesystem
trivy fs /path/to/project

# Scan Kubernetes configs
trivy config ./manifests/

# Scan running cluster
trivy k8s cluster --report summary

# Scan specific namespace
trivy k8s --namespace production all

# Output as JSON
trivy image --format json -o results.json nginx:latest
```

### Grype (Anchore)
**Purpose**: Vulnerability scanner for container images

```bash
# Scan image
grype nginx:latest

# Output specific format
grype nginx:latest -o json

# Scan from SBOM
grype sbom:./sbom.json
```

---

## Compliance Scanning Tools

### kube-bench
**Purpose**: CIS Kubernetes Benchmark compliance checker

```bash
# Run all checks
kube-bench run

# Run on master/control plane
kube-bench run --targets=master

# Run on worker node
kube-bench run --targets=node

# Run specific checks
kube-bench run --check=1.1.1,1.1.2

# Output as JSON
kube-bench run --json > results.json

# Run as a Job in cluster
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

### Kubescape
**Purpose**: Multi-framework security scanner (NSA, CIS, MITRE)

```bash
# Scan with NSA framework
kubescape scan framework nsa

# Scan with CIS benchmark
kubescape scan framework cis-v1.23-t1.0.1

# Scan with MITRE ATT&CK
kubescape scan framework mitre

# Scan all frameworks
kubescape scan framework all

# Scan specific namespace
kubescape scan --include-namespaces production

# Scan YAML files
kubescape scan *.yaml

# Set compliance threshold
kubescape scan framework nsa --compliance-threshold 80

# Output formats
kubescape scan --format json --output results.json
kubescape scan --format html --output report.html
```

### Polaris
**Purpose**: Kubernetes best practices validator

```bash
# Run dashboard
polaris dashboard --port 8080

# Audit cluster
polaris audit

# Audit with specific format
polaris audit --format yaml
polaris audit --format json

# Audit specific namespace
polaris audit --namespace production

# Run as webhook (admission controller)
polaris webhook
```

---

## Runtime Security Tools

### Falco
**Purpose**: Runtime threat detection and syscall monitoring

```bash
# Run Falco
falco

# Run with specific rules file
falco -r /etc/falco/falco_rules.yaml

# Run with custom config
falco -c /etc/falco/falco.yaml

# Output to JSON
falco -o json_output=true

# Test rule parsing
falco --validate /etc/falco/rules.d/

# List available fields
falco --list
```

**Common Falco Rules to Know**:
```yaml
# Terminal shell in container
- rule: Terminal shell in container
  condition: spawned_process and container and shell_procs
  output: Shell in container (user=%user.name container=%container.name)
  priority: WARNING

# Privileged container started
- rule: Launch Privileged Container
  condition: container.privileged=true
  output: Privileged container (container=%container.name image=%container.image.repository)
  priority: WARNING

# Sensitive file read
- rule: Read sensitive file
  condition: open_read and sensitive_files
  output: Sensitive file opened (file=%fd.name)
  priority: WARNING
```

---

## Image Signing Tools

### Cosign (Sigstore)
**Purpose**: Container image signing and verification

```bash
# Generate key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key myregistry/myimage:tag

# Sign with OIDC (keyless)
cosign sign myregistry/myimage:tag

# Verify signature
cosign verify --key cosign.pub myregistry/myimage:tag

# Verify keyless signature
cosign verify myregistry/myimage:tag \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://accounts.google.com

# Attach attestation
cosign attest --key cosign.key --predicate sbom.json myregistry/myimage:tag
```

---

## SBOM Generation Tools

### Syft
**Purpose**: Generate Software Bill of Materials

```bash
# Generate SBOM for image
syft nginx:latest

# Output as CycloneDX JSON
syft nginx:latest -o cyclonedx-json > sbom.json

# Output as SPDX
syft nginx:latest -o spdx-json > sbom.spdx.json

# Scan directory
syft dir:/path/to/project

# Scan archive
syft archive.tar.gz
```

---

## Policy Engines

### OPA/Gatekeeper
**Purpose**: Policy enforcement using Rego language

```bash
# Install Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# Check Gatekeeper status
kubectl get pods -n gatekeeper-system

# List constraint templates
kubectl get constrainttemplates

# List constraints
kubectl get constraints

# Test policy with OPA
opa eval --data policy.rego --input input.json "data.example.allow"
```

**Example Gatekeeper Constraint**:
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        provided := {l | input.review.object.metadata.labels[l]}
        required := {l | l := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing labels: %v", [missing])
      }
```

### Kyverno
**Purpose**: Kubernetes-native policy management (YAML)

```bash
# Install Kyverno
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml

# Check Kyverno status
kubectl get pods -n kyverno

# List policies
kubectl get clusterpolicies
kubectl get policies -A

# Test policy
kyverno apply policy.yaml --resource resource.yaml
```

**Example Kyverno Policy**:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-team-label
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Label 'team' is required"
      pattern:
        metadata:
          labels:
            team: "?*"
```

---

## Certificate Management

### cert-manager
**Purpose**: Automated TLS certificate management

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Check status
kubectl get pods -n cert-manager

# List issuers
kubectl get issuers -A
kubectl get clusterissuers

# List certificates
kubectl get certificates -A

# Describe certificate
kubectl describe certificate my-cert -n my-namespace
```

---

## Kubernetes Security Commands

### RBAC Commands
```bash
# Check if user can perform action
kubectl auth can-i create pods
kubectl auth can-i delete secrets --as=jane
kubectl auth can-i '*' '*' --as=system:serviceaccount:default:my-sa

# List what user can do
kubectl auth can-i --list --as=jane

# List roles
kubectl get roles -A
kubectl get clusterroles

# List bindings
kubectl get rolebindings -A
kubectl get clusterrolebindings

# Describe role permissions
kubectl describe role pod-reader -n default
```

### Audit and Debugging
```bash
# Get events
kubectl get events --sort-by='.lastTimestamp'

# Describe pod for security context
kubectl describe pod my-pod | grep -A 20 "Security Context"

# Check pod security
kubectl get pod my-pod -o jsonpath='{.spec.securityContext}'

# View secrets (encoded)
kubectl get secret my-secret -o yaml

# Decode secret
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# Check network policies
kubectl get networkpolicies -A

# Describe network policy
kubectl describe networkpolicy my-policy
```

### Pod Security Admission
```bash
# Label namespace for PSA enforcement
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# Check namespace labels
kubectl get namespace production --show-labels

# Dry-run to test PSA
kubectl label --dry-run=server namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

---

## Tool Comparison Quick Reference

| Tool | Primary Purpose | Language/Format |
|------|-----------------|-----------------|
| **Trivy** | Vulnerability scanning | N/A |
| **kube-bench** | CIS Benchmark | N/A |
| **Kubescape** | Multi-framework scanning | N/A |
| **Falco** | Runtime security | YAML rules |
| **OPA/Gatekeeper** | Policy enforcement | Rego |
| **Kyverno** | Policy enforcement | YAML |
| **Cosign** | Image signing | N/A |
| **Syft** | SBOM generation | N/A |
| **cert-manager** | Certificate automation | YAML |
| **Polaris** | Best practices | N/A |

---

## CNCF Project Status

| Tool | CNCF Status |
|------|-------------|
| Falco | **Graduated** |
| OPA | **Graduated** |
| SPIFFE/SPIRE | **Graduated** |
| Notary/TUF | **Graduated** |
| cert-manager | **Incubating** |
| Kyverno | **Incubating** |
| in-toto | **Incubating** |

---

## Quick Install Commands

```bash
# Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Kubescape
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | bash

# Falco (Helm)
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco

# Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# Kyverno
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml

# cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Cosign
curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# Syft
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh
```

---