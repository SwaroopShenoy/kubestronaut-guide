# RBAC Deep Hardening Masterclass
## CKS Deep Dive - Least Privilege for System Components

---

# Section 1: RBAC Fundamentals Review

## Quick Recap

**RBAC** = Role-Based Access Control. Who (ServiceAccount/User) can do what (Verb) on what (Resource).

**Three pieces**:
- **Role/ClusterRole**: Defines permissions (can create pods? delete secrets?)
- **RoleBinding/ClusterRoleBinding**: Assigns role to user/service account
- **ServiceAccount**: Identity for processes running in pods

---

# Section 2: The Problem (Why Deep Hardening Matters)

## Default Kubernetes: Too Permissive

```bash
# Default: system:unauthenticated can do... nothing (good)
# But: system:serviceaccount:default:default can do... EVERYTHING (bad!)

k auth can-i create pods --as=system:serviceaccount:default:default
# YES! It shouldn't be.

k auth can-i delete secrets --as=system:serviceaccount:default:default
# YES! Way too much power.
```

**Attack scenario**:
```
Attacker compromises Pod running as default:default
    ↓
Pod has unlimited permissions
    ↓
Attacker can: create pods, steal secrets, modify RBAC, escape cluster
    ↓
Full cluster compromise
```

**Solution**: Bind minimal permissions to each ServiceAccount.

---

# Section 3: Least Privilege Design

## Principle: Every SA Gets Only What It Needs

```
Application Pod
  ├─ ServiceAccount: app
  │   └─ Can: READ pods in same namespace
  │          READ configmaps in same namespace
  │
  ├─ ServiceAccount: monitoring
  │   └─ Can: READ pods in all namespaces (for monitoring)
  │          READ metrics endpoints
  │
  └─ ServiceAccount: admin
      └─ Can: (nothing!) Delete if not needed
```

---

# Section 4: Real Exam Scenarios

## Scenario 1: Audit Current RBAC

**Question**: What can the default ServiceAccount do? Too much?

```bash
# List all ClusterRoleBindings
k get clusterrolebindings

# Find dangerous bindings
k get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin")'

# Who has cluster-admin?
k get clusterrolebindings -o wide | grep cluster-admin

# Audit: cluster-admin should ONLY be bound to:
# - system:masters (users, not serviceaccounts)
# NOT: default:default, or regular users
```

## Scenario 2: Fix Overpermissioned ServiceAccount

**Question**: ServiceAccount "app" can do too much. Restrict it.

```bash
# 1. Check current permissions
k describe sa app -n production

# 2. Find all RoleBindings for this SA
k get rolebindings -n production -o json | jq '.items[] | select(.subjects[].name=="app")'

# 3. Delete overpermissioned bindings
k delete rolebinding app-admin -n production

# 4. Create restrictive role
k apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-read-only
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch"]
EOF

# 5. Bind it
k apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-read-only
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-read-only
subjects:
- kind: ServiceAccount
  name: app
  namespace: production
EOF

# 6. Verify
k auth can-i get pods --as=system:serviceaccount:production:app -n production
# YES (allowed)

k auth can-i delete pods --as=system:serviceaccount:production:app -n production
# NO (denied)
```

## Scenario 3: System Component Hardening

**Question**: kubelet ServiceAccount has too much power. Restrict it.

```bash
# Current: system:serviceaccount:kube-system:kubelet can do everything

# Better: kubelet only needs to:
# - Read pods in all namespaces
# - Read config maps in kube-system
# - Create/update pod status

k apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubelet-limited
rules:
# Read pods (needed to schedule)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Update pod status (needed to report)
- apiGroups: [""]
  resources: ["pods/status"]
  verbs: ["update", "patch"]

# Read configmaps (needed for config)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
  namespaces: ["kube-system"]

# Read secrets only if needed
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  namespaces: ["kube-system"]
EOF

# Bind to kubelet SA
k create clusterrolebinding kubelet-limited \
  --clusterrole=kubelet-limited \
  --serviceaccount=kube-system:kubelet
```

## Scenario 4: Audit RBAC Changes

**Question**: Who modified RBAC recently? Find suspicious changes.

```bash
# Use audit logs with jq
cat /var/log/kubernetes/audit/audit.log | jq '.[] | select(.objectRef.resource | contains("role")) | {time: .requestReceivedTimestamp, user: .user.username, verb: .verb, resource: .objectRef.resource, name: .objectRef.name}'

# Output shows: who, when, what role was changed
```

## Scenario 5: Deny Dangerous Permissions

**Question**: Prevent anyone from using wildcards (*) in RBAC rules.

Use OPA Gatekeeper (from earlier masterclass):

```rego
package k8sdenywildcards

violation[{"msg": msg}] {
  rule := input.review.object.rules[_]
  
  # Check for wildcard verbs
  verb := rule.verbs[_]
  verb == "*"
  
  msg := "Wildcard verbs (*) not allowed in RBAC rules"
}
```

---

# Section 5: RBAC Best Practices

## 1. Never Use Wildcards

❌ **WRONG**:
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

✅ **RIGHT**:
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

## 2. Bind to ServiceAccounts, Not Users

❌ **WRONG**:
```yaml
subjects:
- kind: User
  name: alice
```

✅ **RIGHT** (in namespaces):
```yaml
subjects:
- kind: ServiceAccount
  name: app
  namespace: production
```

## 3. Use Namespaced Roles When Possible

❌ **WRONG** (grants cluster-wide):
```yaml
kind: ClusterRole
```

✅ **RIGHT** (limited to namespace):
```yaml
kind: Role
metadata:
  namespace: production
```

## 4. Audit Frequently

```bash
# Weekly audit
k get clusterrolebindings -o wide | grep -v system
k get rolebindings -A -o wide | grep -v system

# Remove unnecessary bindings
k delete clusterrolebinding <name>
```

## 5. Deny cluster-admin for Regular Users

```bash
# Only system:masters should have cluster-admin
k get clusterrolebindings cluster-admin -o wide

# If regular users found: remove them
k delete clusterrolebinding cluster-admin
k create clusterrolebinding cluster-admin \
  --clusterrole=cluster-admin \
  --group=system:masters
```

---

# Section 6: Cheat Sheet

## Quick Audit Commands

```bash
# Who has cluster-admin?
k get clusterrolebindings cluster-admin -o wide

# All dangerous bindings
k get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name | contains("admin"))'

# All RoleBindings for specific SA
k get rolebindings -A -o json | jq '.items[] | select(.subjects[].name=="default")'

# Check if user can do action
k auth can-i create pods --as=alice -n production

# Check if SA can do action
k auth can-i delete secrets --as=system:serviceaccount:default:default
```

## Quick Fix: Restrictive Role Template

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-minimal
  namespace: default
rules:
# Read pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

# Read configmaps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-minimal
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-minimal
subjects:
- kind: ServiceAccount
  name: app
  namespace: default
```

---

# Pre-Exam Checklist

- ✅ Understand: RBAC = Role + RoleBinding + ServiceAccount
- ✅ Audit current permissions (who has what?)
- ✅ Identify overpermissioned bindings (cluster-admin, wildcards)
- ✅ Create restrictive roles (only what's needed)
- ✅ Use `k auth can-i` to verify
- ✅ Query audit logs for RBAC changes
- ✅ Speed: Should take <10 min per audit + fix

