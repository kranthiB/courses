# KCSA Exam Traps and Gotchas

## 🚨 CRITICAL: These Trick Questions Catch Many People!

---

## TRAP #1: Secrets Are NOT Encrypted by Default

**Wrong Answer Trap**: "Kubernetes Secrets are encrypted"

**Correct Answer**: Secrets are **base64 ENCODED**, not encrypted!

```bash
# Anyone can decode a secret
echo "cGFzc3dvcmQ=" | base64 -d
# Output: password
```

**What you need to know**:
- Base64 is encoding, NOT encryption
- Enable encryption at rest with `--encryption-provider-config`
- Use external secret managers (Vault, AWS Secrets Manager) for production

---

## TRAP #2: PodSecurityPolicy is REMOVED

**Wrong Answer Trap**: "Use PodSecurityPolicy to restrict pods"

**Correct Answer**: PSP was **removed in Kubernetes 1.25**. Use Pod Security Admission (PSA).

| Old (Removed) | New (Current) |
|---------------|---------------|
| PodSecurityPolicy | Pod Security Admission |
| Custom policies | Three levels: Privileged, Baseline, Restricted |

**Remember**: If you see PSP in an answer, it's likely wrong unless asking about deprecated features.

---

## TRAP #3: Default Network Policy is ALLOW ALL

**Wrong Answer Trap**: "Kubernetes denies traffic by default"

**Correct Answer**: By default, **ALL pod-to-pod traffic is ALLOWED**

**To deny traffic, you must**:
```yaml
# Explicit deny-all policy required
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## TRAP #4: STRIDE Order Matters

**Wrong Answer Trap**: Getting the STRIDE letters wrong

**Correct Answer**: **S**poofing, **T**ampering, **R**epudiation, **I**nformation Disclosure, **D**enial of Service, **E**levation of Privilege

**Mnemonic**: "**S**ome **T**hreats **R**equire **I**mmediate **D**efensive **E**fforts"

---

## TRAP #5: kube-bench vs Kubescape

**Wrong Answer Trap**: Confusing what each tool does

| Tool | Purpose |
|------|---------|
| **kube-bench** | CIS Benchmark ONLY |
| **Kubescape** | Multiple frameworks (NSA, CIS, MITRE) |
| **Trivy** | Vulnerabilities + Misconfigurations |
| **Falco** | Runtime threat detection |

---

## TRAP #6: Admission Controller Order

**Wrong Answer Trap**: "Validating webhooks run first"

**Correct Answer**: **Mutating → then → Validating**

```
Request → Authentication → Authorization → Mutating → Validating → etcd
```

**Why it matters**: Mutating webhooks can change the request before validation.

---

## TRAP #7: mTLS Provides TWO Things

**Wrong Answer Trap**: "mTLS only encrypts traffic"

**Correct Answer**: mTLS provides:
1. **Mutual Authentication** (both parties verify each other)
2. **Encryption** (traffic is encrypted)

Regular TLS only authenticates the server. mTLS authenticates BOTH client and server.

---

## TRAP #8: PSS Levels Are NOT "High/Medium/Low"

**Wrong Answer Trap**: Using generic security level names

**Correct Answer**: Pod Security Standards levels are:
- **Privileged** (unrestricted)
- **Baseline** (minimal restrictions)
- **Restricted** (hardened)

NOT: High, Medium, Low or Strict, Standard, Permissive

---

## TRAP #9: 4Cs Order (Outer to Inner)

**Wrong Answer Trap**: Getting the order backwards

**Correct Answer**: **Cloud → Cluster → Container → Code**

```
CLOUD (outermost)
  └─► CLUSTER
       └─► CONTAINER
            └─► CODE (innermost)
```

Each layer depends on the security of the layer outside it.

---

## TRAP #10: kubelet Read-Only Port

**Wrong Answer Trap**: "Port 10250 should be disabled"

**Correct Answer**: 
- **Port 10255** is the read-only port (DISABLE THIS)
- Port 10250 is the main kubelet API (needs authentication)

```yaml
# Disable read-only port
--read-only-port=0
```

---

## TRAP #11: OPA vs Kyverno Language

**Wrong Answer Trap**: Confusing which uses which language

| Tool | Language |
|------|----------|
| **OPA/Gatekeeper** | Rego |
| **Kyverno** | YAML (Kubernetes-native) |

**Memory Trick**: Kyverno = Kubernetes native = YAML

---

## TRAP #12: containerd is Now Default (Not Docker)

**Wrong Answer Trap**: "Docker is the default container runtime"

**Correct Answer**: **containerd** is the default since Kubernetes 1.24

- dockershim was REMOVED in v1.24
- Docker images still work (OCI compatible)
- containerd or CRI-O are current options

---

## TRAP #13: NIST CSF Has 5 Functions (Not 4 or 6)

**Wrong Answer Trap**: Forgetting or adding functions

**Correct Answer**: **I**dentify, **P**rotect, **D**etect, **R**espond, **R**ecover

**Mnemonic**: "**I** **P**ractice **D**aily **R**esilience **R**outines"

---

## TRAP #14: SLSA is About Supply Chain (Not Secrets)

**Wrong Answer Trap**: Thinking SLSA is a secrets framework

**Correct Answer**: SLSA = **Supply-chain Levels for Software Artifacts**

| Level | Focus |
|-------|-------|
| SLSA 1 | Documentation |
| SLSA 2 | Tamper-resistant builds |
| SLSA 3 | Hardened builds |
| SLSA 4 | Two-party review, hermetic builds |

---

## TRAP #15: Privileged Pod vs Individual Capabilities

**Wrong Answer Trap**: Confusing privileged mode with specific capabilities

| Setting | Effect |
|---------|--------|
| `privileged: true` | ALL capabilities, full host access |
| `capabilities.add: [SYS_ADMIN]` | Only specific capability |

**`privileged: true`** is the most dangerous - never use in production!

---

## TRAP #16: SBOM vs SAST vs SCA

**Wrong Answer Trap**: Confusing these acronyms

| Acronym | Meaning | Purpose |
|---------|---------|---------|
| **SBOM** | Software Bill of Materials | Inventory of components |
| **SAST** | Static Application Security Testing | Code analysis |
| **SCA** | Software Composition Analysis | Dependency scanning |

---

## TRAP #17: Cosign vs Notary

**Wrong Answer Trap**: Not knowing which is current

**Correct Answer**: Both are valid, but:
- **Cosign** (Sigstore) - newer, simpler, more common
- **Notary** (TUF) - older, more complex

If the question mentions "Sigstore", the answer is **Cosign**.

---

## TRAP #18: ClusterRole vs Role Scope

**Wrong Answer Trap**: Thinking they're interchangeable

| Resource | Scope |
|----------|-------|
| **Role** | Namespace only |
| **ClusterRole** | Cluster-wide OR can be bound to namespace |
| **RoleBinding** | Binds role in a namespace |
| **ClusterRoleBinding** | Binds role cluster-wide |

**Key**: ClusterRole + RoleBinding = ClusterRole permissions in ONE namespace

---

## TRAP #19: etcd Stores EVERYTHING

**Wrong Answer Trap**: Thinking etcd only stores configuration

**Correct Answer**: etcd stores ALL cluster state:
- Secrets (readable if you have etcd access!)
- ConfigMaps
- Pod specs
- Service accounts
- RBAC policies
- Everything!

**This is why etcd security is CRITICAL**

---

## TRAP #20: Cloud Metadata API IP

**Wrong Answer Trap**: Not knowing the specific IP

**Correct Answer**: **169.254.169.254**

This IP is used by ALL major cloud providers (AWS, GCP, Azure) for instance metadata. Block it with NetworkPolicy to prevent credential theft.

---

## Quick Reference: Common Confusions

| Often Confused | Difference |
|----------------|------------|
| Encryption vs Encoding | Encryption needs a key; encoding doesn't |
| AuthN vs AuthZ | AuthN = who are you; AuthZ = what can you do |
| Role vs ClusterRole | Namespace vs cluster scope |
| Mutating vs Validating | Modify vs accept/reject |
| SAST vs DAST | Static (code) vs Dynamic (running) |
| Falco vs Trivy | Runtime detection vs vulnerability scanning |
| CIS vs NIST | Benchmark (config) vs Framework (process) |

---

## Exam Day Memory Checklist

### Before Answering, Ask Yourself:

1. **Is this about CURRENT Kubernetes?** (PSP is gone!)
2. **What's the DEFAULT behavior?** (Network = allow all, Secrets = encoded)
3. **What's the SCOPE?** (Namespace vs Cluster)
4. **What LAYER of 4Cs?** (Cloud → Cluster → Container → Code)
5. **What TOOL does what?** (kube-bench = CIS, Kubescape = multiple)

---

## Red Flag Words in Wrong Answers

If you see these, be suspicious:

| Word/Phrase | Why It's Suspicious |
|-------------|---------------------|
| "encrypted by default" | Secrets aren't encrypted by default |
| "PodSecurityPolicy" | Deprecated/removed |
| "Docker runtime" | Not default since 1.24 |
| "denies traffic by default" | Network allows all by default |
| "validating runs first" | Mutating runs first |

---

## Final Tips

1. **Read carefully**: Look for "NOT", "EXCEPT", "DEFAULT"
2. **Eliminate obviously wrong**: Usually 1-2 answers are clearly wrong
3. **Think about security layers**: 4Cs, defense in depth
4. **Remember acronyms**: STRIDE, DREAD, NIST CSF functions
5. **Tools have specific purposes**: Don't confuse kube-bench with Falco

---

**Master these traps = Pass with confidence! 🎯**