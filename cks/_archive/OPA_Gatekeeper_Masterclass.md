# OPA/Gatekeeper Masterclass
## CKS Deep Dive - Policy-as-Code Enforcement

---

# Section 1: OPA/Gatekeeper Fundamentals

## What is OPA?

**OPA** (Open Policy Agent) = Policy engine. Write security rules, OPA enforces them.

**Gatekeeper** = OPA integrated into Kubernetes as admission controller.

**Think of it as**: Security gatekeeper that says "YES" or "NO" to Pod/Deployment deployments based on YOUR rules.

**Default Kubernetes**: No built-in way to enforce "all Pods must have resource limits" or "no root containers" (SecurityContext does it but not at admission time).

**With Gatekeeper**: Write policy, auto-reject non-compliant Pods. ✅

---

## When Does It Trigger?

Gatekeeper intercepts at **admission time** (before creation):

```
User creates Pod
    ↓
Pod reaches API server
    ↓
Gatekeeper checks policies
    ↓
✅ ACCEPT: Create Pod
❌ REJECT: Deny creation, show error
```

**Advantage**: Policy enforcement BEFORE deployment (not after).

---

## OPA vs Kubernetes Security Features

| Feature | How It Works | When It Applies |
|---|---|---|
| **SecurityContext** | Pod spec sets runAsUser, etc. | Container level, after admission |
| **Pod Security Standards** | Labels on namespace enforce level | At admission, but limited to preset levels |
| **NetworkPolicy** | Traffic rules between pods | Network level, runtime |
| **OPA/Gatekeeper** | Custom policies, fully programmable | Admission time, ANY rule you write |

**Why use Gatekeeper**: Maximum flexibility. Write ANY rule.

---

# Section 2: Gatekeeper Architecture

## Components

### 1. ConstraintTemplate

**Defines the rule logic** (the "what" to check).

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresourcelimits
spec:
  # 1. Define CRD (creates K8sRequiredResourceLimits custom resource)
  crd:
    spec:
      names:
        kind: K8sRequiredResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            limits:
              type: array
              items:
                type: string
  
  # 2. Write the rule (Rego language)
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredresourcelimits
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits
        msg := sprintf("Container %v must have resource limits", [container.name])
      }
```

### 2. Constraint

**Applies the rule** (the "how" to apply).

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceLimits
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
    excludedNamespaces:
    - kube-system
    - kube-public
  parameters:
    limits: ["cpu", "memory"]
```

**Flow**: ConstraintTemplate + Constraint = Policy enforcement.

---

# Section 3: Installing Gatekeeper

## Step 1: Install Gatekeeper

```bash
# Using kubectl (official repo)
k apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# Verify
k get deployment -n gatekeeper-system
k get pods -n gatekeeper-system

# Should see: gatekeeper-audit, gatekeeper-controller-manager, etc.
```

## Step 2: Wait for Webhook to Be Ready

```bash
# Gatekeeper installs a webhook (admission controller)
# It needs time to start

k get validatingwebhookconfigurations | grep gatekeeper

# Wait until webhook is ready (may take 30 seconds)
```

## Step 3: Verify Gatekeeper is Working

```bash
# Create test Constraint
k apply -f - <<EOF
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
        not input.review.object.metadata.labels.app
        msg := "Pod must have label 'app'"
      }
EOF

# Apply Constraint
k apply -f - <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
EOF

# Test: Try to create pod WITHOUT label
k run test --image=nginx
# Error: Pod must have label 'app'

# Test: Create pod WITH label
k run test --image=nginx --labels app=web
# Success!

# Cleanup
k delete pod test
```

---

# Section 4: Writing Rego Policies

## Rego Language Basics

Rego (pronounced "ray-go") is OPA's policy language. It's **declarative** (not imperative).

### Basic Syntax

```rego
package k8srequiredlimits

# Define a violation
violation[{"msg": msg}] {
  # This block runs if all conditions are true
  condition1 is true AND
  condition2 is true
  # → Add violation with message
  msg := "Your error message"
}
```

### Common Patterns

#### Pattern 1: Check Container Limits

```rego
package k8srequiredresourcelimits

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]  # Loop all containers
  not container.resources.limits                        # No limits defined
  msg := sprintf("Container %v missing limits", [container.name])
}
```

#### Pattern 2: Enforce Non-Root

```rego
package k8srequirednonroot

violation[{"msg": msg}] {
  pod := input.review.object
  not pod.spec.securityContext.runAsNonRoot           # Not set to non-root
  msg := "Pod must have runAsNonRoot: true"
}
```

#### Pattern 3: Block Privileged Pods

```rego
package k8sblockedprivileged

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  container.securityContext.privileged                # Is privileged?
  msg := sprintf("Container %v cannot be privileged", [container.name])
}
```

#### Pattern 4: Require Specific Image Registry

```rego
package k8srequiredimageregistry

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  image := container.image
  not startswith(image, "myregistry.azurecr.io/")     # Wrong registry
  msg := sprintf("Image must come from myregistry.azurecr.io/, got %v", [image])
}
```

#### Pattern 5: Deny Certain Namespaces

```rego
package k8sdenysensitivenamespace

violation[{"msg": msg}] {
  namespace := input.review.object.metadata.namespace
  namespace in ("kube-system", "kube-public")         # Restricted namespaces
  msg := sprintf("Cannot deploy to namespace %v", [namespace])
}
```

#### Pattern 6: Require Multiple Conditions

```rego
package k8srequiredsecure

violation[{"msg": msg}] {
  pod := input.review.object
  
  # ALL conditions must be true for violation
  not pod.spec.securityContext.runAsNonRoot AND
  not pod.spec.securityContext.readOnlyRootFilesystem AND
  not pod.spec.securityContext.capabilities.drop
  
  msg := "Pod must have: runAsNonRoot, readOnlyRootFilesystem, dropped capabilities"
}
```

---

# Section 5: Real Exam Scenarios

## Scenario 1: Enforce Resource Limits

**Question**: Create policy to reject Pods without CPU/memory limits.

```bash
# 1. Create ConstraintTemplate
k apply -f - <<EOF
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresourcelimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResourceLimits
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredresourcelimits
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits
        msg := sprintf("Container %v missing resource limits", [container.name])
      }
EOF

# 2. Apply Constraint
k apply -f - <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceLimits
metadata:
  name: require-limits
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
EOF

# 3. Test
# Create Deployment WITHOUT limits → REJECTED
# Create Deployment WITH limits → ACCEPTED
```

## Scenario 2: Block Privileged Containers

**Question**: Policy to reject any privileged containers.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockedprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sBlockedPrivileged
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sblockedprivileged
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        container.securityContext.privileged
        msg := sprintf("Container %v cannot be privileged", [container.name])
      }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockedPrivileged
metadata:
  name: block-privileged
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
```

## Scenario 3: Enforce Image Registry

**Question**: Only allow images from internal registry.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredimageregistry
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredImageRegistry
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedRegistries:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredimageregistry
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        image := container.image
        
        allowed := false
        for registry in input.parameters.allowedRegistries {
          startswith(image, registry) && (allowed := true)
        }
        
        not allowed
        msg := sprintf("Image %v not from allowed registry", [image])
      }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredImageRegistry
metadata:
  name: require-internal-registry
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
  parameters:
    allowedRegistries:
    - "gcr.io/my-org/"
    - "ecr.aws/my-org/"
```

## Scenario 4: Require Non-Root + Read-Only Filesystem

**Question**: Enforce secure SecurityContext.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredsecuritycontext
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        
        # Must have runAsNonRoot
        not container.securityContext.runAsNonRoot
        msg := sprintf("Container %v must have runAsNonRoot: true", [container.name])
      }
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        
        # Must have readOnlyRootFilesystem
        not container.securityContext.readOnlyRootFilesystem
        msg := sprintf("Container %v must have readOnlyRootFilesystem: true", [container.name])
      }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: require-secure-context
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
    excludedNamespaces:
    - kube-system
```

## Scenario 5: Audit Mode (Warn, Don't Block)

**Question**: Log violations but allow (for testing policies).

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceLimits
metadata:
  name: require-limits-audit
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
  
  # Audit mode: log violations but don't block
  enforcementAction: audit  # Options: deny (default), audit, dryrun
```

**Results**: Violations logged but Pods created successfully. Useful for testing.

---

# Section 6: Debugging Gatekeeper

### Policy Not Enforcing?

```bash
# 1. Check if Gatekeeper running
k get pods -n gatekeeper-system

# 2. Check webhook registered
k get validatingwebhookconfigurations | grep gatekeeper

# 3. View webhook logs
k logs -f pod/gatekeeper-controller-manager-xxx -n gatekeeper-system

# 4. Check Constraint is applied
k get K8sRequiredResourceLimits

# 5. Check Constraint status
k describe K8sRequiredResourceLimits require-limits
# Should show: "Number of violations: X"
```

### Policy Rejecting Valid Pods?

```bash
# Describe the Constraint
k describe K8sRequiredResourceLimits require-limits

# Read the rego rule carefully
# Common mistakes:
# - Wrong field path (container.resources vs pod.spec.resources)
# - Not checking all containers
# - Logic error in condition

# Test policy without enforcement (audit mode)
k patch K8sRequiredResourceLimits require-limits \
  --type=merge \
  -p '{"spec":{"enforcementAction":"audit"}}'

# This logs violations without blocking
```

---

# Section 7: Cheat Sheet

## Quick Install

```bash
# Deploy Gatekeeper
k apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# Verify
k get pods -n gatekeeper-system
```

## Quick ConstraintTemplate (Copy-Paste)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresourcelimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResourceLimits
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredresourcelimits
      
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.limits
        msg := sprintf("Container %v must have limits", [container.name])
      }
```

## Quick Constraint (Copy-Paste)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceLimits
metadata:
  name: require-limits
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
    excludedNamespaces:
    - kube-system
    - kube-public
```

## Common Rego Patterns

```rego
# Loop containers
container := input.review.object.spec.containers[_]

# Check field exists
container.resources.limits

# Negate
not container.resources.limits

# String match
startswith(image, "gcr.io/")
endswith(image, ":latest")

# Array membership
verb in ("create", "update", "patch")

# Conditional message
msg := sprintf("Container %v error", [container.name])

# Multiple violations per container
violation[{"msg": msg1}] { ... }
violation[{"msg": msg2}] { ... }
```

## Debugging Commands

```bash
# Check ConstraintTemplate loaded
k get constrainttemplates

# Check Constraint created
k get K8sRequiredResourceLimits

# View Constraint details
k describe K8sRequiredResourceLimits require-limits

# Check violations
k get K8sRequiredResourceLimits require-limits -o yaml | grep -A 5 "status:"

# Webhook logs
k logs -n gatekeeper-system pod/gatekeeper-controller-manager-xxx -f

# Test audit mode
k patch K8sRequiredResourceLimits require-limits --type=merge -p '{"spec":{"enforcementAction":"audit"}}'
```

---

# Pre-Exam Checklist

- ✅ Understand: ConstraintTemplate defines rule, Constraint applies it
- ✅ Install Gatekeeper
- ✅ Write simple rego policy (require field, deny value)
- ✅ Create ConstraintTemplate + Constraint pair
- ✅ Test: Policy blocks invalid Pods, allows valid ones
- ✅ Audit mode: Test policy without enforcement
- ✅ Debug: Read webhook logs, check Constraint status
- ✅ Speed: Should take <15 min to write + deploy policy

---

# Speed Targets for Exam

- **Install Gatekeeper**: <2 min
- **Write ConstraintTemplate**: <5 min (copy from template, modify rego)
- **Create Constraint**: <2 min
- **Test policy**: <3 min
- **Debug failing policy**: <5 min

If you hit these speeds, you OWN this topic. 💪

