# Rego Masterclass
## OPA Policy Language - Write Security Rules Like Code

---

# What is Rego?

**Rego** = Open Policy Agent's policy language. Write security rules declaratively.

Think of it as: **Prolog-like language for "what should be allowed?"**

**Key difference from other languages**: Not imperative (if → do). Declarative (facts → conclusions).

---

# Rego Fundamentals

## 1. Basic Rule Structure

```rego
package mypolicy

# If condition is true, this violation exists
violation[{"msg": msg}] {
  condition1 is true
  condition2 is true
  msg := "Error message"
}
```

**How it works**:
- Rule runs against input data
- If ALL conditions in block are true → violation added
- If ANY condition fails → rule skips

## 2. Basic Syntax

```rego
package k8srequiredlimits

# Access input data
input.review.object.spec.containers[0].name

# Loop through array
container := input.review.object.spec.containers[_]
# (_) means "any index" - iterate all

# Check if field exists
container.resources.limits

# Negate (NOT)
not container.resources.limits

# Assign variable
msg := "Container missing limits"

# String template
msg := sprintf("Container %v missing limits", [container.name])
```

---

# Common Rego Patterns for CKS

## Pattern 1: Loop and Check Each Container

```rego
package k8srequiredlimits

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]  # Loop ALL containers
  not container.resources.limits                        # No limits?
  msg := sprintf("Container %v missing limits", [container.name])
}
```

**How it works**:
- `container := ... [_]` iterates through ALL containers
- For EACH container, if no limits → violation
- If pod has 3 containers, can generate UP TO 3 violations

## Pattern 2: Check Pod-Level Field

```rego
package k8srequirednonroot

violation[{"msg": msg}] {
  pod := input.review.object
  not pod.spec.securityContext.runAsNonRoot
  msg := "Pod must have runAsNonRoot: true"
}
```

**Note**: Pod-level check (not looping containers).

## Pattern 3: Block If Condition Met

```rego
package k8sblockedprivileged

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  container.securityContext.privileged              # Is it privileged?
  msg := sprintf("Container %v cannot be privileged", [container.name])
}
```

## Pattern 4: Check Against Parameter List

```rego
package k8srequiredimageregistry

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  image := container.image
  
  # Check if image starts with allowed registry
  allowed := false
  for registry in input.parameters.allowedRegistries {
    startswith(image, registry) && (allowed := true)
  }
  
  not allowed
  msg := sprintf("Image %v not from allowed registry", [image])
}
```

## Pattern 5: Multiple Violations (All Conditions)

```rego
package k8srequiredsecure

# Violation 1: No runAsNonRoot
violation[{"msg": msg}] {
  pod := input.review.object
  not pod.spec.securityContext.runAsNonRoot
  msg := "Must have runAsNonRoot: true"
}

# Violation 2: No readOnlyRootFilesystem
violation[{"msg": msg}] {
  pod := input.review.object
  not pod.spec.securityContext.readOnlyRootFilesystem
  msg := "Must have readOnlyRootFilesystem: true"
}

# Violation 3: No dropped capabilities
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.securityContext.capabilities.drop
  msg := sprintf("Container %v must drop ALL capabilities", [container.name])
}
```

## Pattern 6: Complex Condition (Multiple AND)

```rego
package k8srequiredlabels

violation[{"msg": msg}] {
  pod := input.review.object
  namespace := pod.metadata.namespace
  
  # All conditions must be true
  namespace != "kube-system" AND
  namespace != "kube-public" AND
  not pod.metadata.labels.app AND
  not pod.metadata.labels.team
  
  msg := "Pod must have 'app' or 'team' label"
}
```

## Pattern 7: Condition (IF-ELSE Logic)

```rego
package k8sconditional

violation[{"msg": msg}] {
  pod := input.review.object
  kind := pod.kind
  
  # Different rules for different kinds
  kind == "Pod" {
    # Pod-specific check
    not pod.spec.securityContext.runAsNonRoot
    msg := "Pod must be non-root"
  }
  
  kind == "Deployment" {
    # Deployment-specific check
    not pod.spec.template.spec.securityContext.runAsNonRoot
    msg := "Deployment must have non-root pods"
  }
}
```

---

# Rego String & Math Operations

## String Functions

```rego
startswith("nginx:latest", "nginx")       # true
endswith("nginx:latest", "latest")        # true
contains("user-admin", "admin")           # true
sprintf("Container %v", ["app"])          # "Container app"
split("a,b,c", ",")                       # ["a", "b", "c"]
```

## Array Functions

```rego
[1, 2, 3] | length                        # 3
[3, 1, 2] | sort                          # [1, 2, 3]
array[_]                                  # Iterate each element
[1, 2] + [3, 4]                           # [1, 2, 3, 4]
```

## Math Operations

```rego
5 > 3                                     # true
10 >= 10                                  # true
3 < 5                                     # true
```

---

# Rego Input Structure (What You Access)

```rego
# In Gatekeeper, input looks like:
input.review.object             # The K8s object (Pod/Deployment/etc)
input.review.operation          # Operation: CREATE, UPDATE, DELETE
input.review.kind               # Kind: Pod, Deployment, Secret
input.review.namespace          # Namespace

input.parameters                # Parameters from Constraint spec

# Inside the object:
input.review.object.kind        # Pod, Deployment, etc
input.review.object.metadata.name
input.review.object.metadata.namespace
input.review.object.metadata.labels
input.review.object.spec        # Spec depends on kind
```

---

# Common Rego Mistakes

## ❌ Mistake 1: Forgot to Iterate Containers

```rego
# WRONG - tries to access spec.containers directly
violation[{"msg": msg}] {
  not input.review.object.spec.containers.resources.limits
  msg := "No limits"
}
# Error: containers is array, can't access directly

# RIGHT - iterate with [_]
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.resources.limits
  msg := "No limits"
}
```

## ❌ Mistake 2: Forgot `not`

```rego
# WRONG - checks if field EXISTS (always true if set)
violation[{"msg": msg}] {
  container.resources.limits
  # This triggers even if limits ARE set!
}

# RIGHT - use `not` to check if NOT set
violation[{"msg": msg}] {
  not container.resources.limits
  msg := "No limits"
}
```

## ❌ Mistake 3: Wrong Nesting for Deployments

```rego
# WRONG - Deployment spec is nested differently
violation[{"msg": msg}] {
  not input.review.object.spec.securityContext.runAsNonRoot
  msg := "Must be non-root"
}
# For Deployment, it's spec.template.spec!

# RIGHT
violation[{"msg": msg}] {
  pod := input.review.object.spec.template  # Get template
  not pod.spec.securityContext.runAsNonRoot
  msg := "Must be non-root"
}
```

## ❌ Mistake 4: Using AND When You Want OR

```rego
# WRONG - all conditions must be true to violate
violation[{"msg": msg}] {
  not container.resources.limits AND
  not container.resources.requests
  # Only violations if BOTH missing
}

# RIGHT - separate violations for each check
violation[{"msg": msg}] {
  not container.resources.limits
  msg := "No limits"
}

violation[{"msg": msg}] {
  not container.resources.requests
  msg := "No requests"
}
# Now violations if EITHER is missing
```

---

# Rego Quick Reference

```rego
# PACKAGES & STRUCTURE
package mypolicy                          # Namespace
violation[{"msg": msg}] { ... }           # Define violation

# ACCESSING DATA
input.review.object.spec.containers       # Access fields
container := input.review.object.spec.containers[_]  # Loop array

# OPERATORS
==, !=, <, >, <=, >=                      # Comparisons
and, or, not                              # Logical
startswith(), endswith(), contains()      # String
sprintf("text %v", [var])                 # String template

# EXAMPLES
not field.exists                          # Field missing
field == "value"                          # Field equals
array[_]                                  # Loop array
"string" in ("a", "b", "string")          # Array membership
```

---

# Pre-Exam Speed Targets

- **Write simple violation rule**: <5 min
- **Add loop through containers**: <2 min
- **Add string matching**: <3 min
- **Debug why rule not firing**: <5 min

