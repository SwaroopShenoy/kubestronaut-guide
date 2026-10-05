# NetworkPolicy Masterclass
## CKS Deep Dive - Zero-Trust Network Security

---

# Part 1: NetworkPolicy Fundamentals

## What is NetworkPolicy?

**NetworkPolicy** = Kubernetes firewall for Pods.

**Default state** (WITHOUT NetworkPolicy):
- All Pods can talk to ALL Pods in cluster (ANY namespace)
- ALL Pods can reach external internet
- ANYONE can reach your Pods (if exposed)

**With NetworkPolicy**:
- Only explicitly allowed traffic flows
- Zero-trust by default

**Reality check**: If your cluster has NO NetworkPolicies, it's WIDE OPEN. CKS assumes you're hardening this.

---

## Key Concepts

### 1. Selectors (Who are we talking about?)

**Pod Selector**: Which Pods does this policy apply to?

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
spec:
  podSelector:
    matchLabels:
      app: backend    # Apply to Pods with label app=backend
```

**Namespace Selector**: Pods from which namespaces?

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: prod   # Allow from Pods in 'prod' namespace
```

**Combined Selectors**: Multiple conditions (AND logic)

```yaml
ingress:
- from:
  - podSelector:
        matchLabels:
          app: frontend    # AND
    namespaceSelector:
      matchLabels:
        environment: prod  # Both must be true
```

### 2. Policy Types

**Ingress**: Control inbound traffic to Pods
**Egress**: Control outbound traffic from Pods

```yaml
spec:
  policyTypes:
  - Ingress      # This policy controls ingress
  - Egress       # This policy controls egress
```

**Critical rule**: If you specify `Egress` in policyTypes, you MUST define egress rules. Otherwise: DEFAULT DENY EGRESS (nothing can leave).

---

## Core Rule: policyTypes Behavior

| policyTypes | Behavior |
|---|---|
| `[Ingress]` | DENY ingress (block inbound), ALLOW all egress (allow outbound) |
| `[Egress]` | ALLOW all ingress (allow inbound), DENY egress (block outbound) |
| `[Ingress, Egress]` | DENY both unless explicitly allowed |
| `[]` (empty) | ALLOW all (no policy) |
| Not specified | ALLOW all (no policy) |

**Exam trap**: Add `Egress` without rules → Pod can't reach anything! 🔥

---

# Part 2: Ingress Rules (Inbound Traffic)

## Ingress Rule Structure

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
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

**Meaning**: "Allow inbound TCP traffic on port 8080 FROM pods labeled app=frontend"

---

## Ingress Selector Patterns

### Pattern 1: Allow from Specific Pod (Same Namespace)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend          # This policy applies to backend Pods
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend     # Allow from frontend Pods in SAME namespace
    
    ports:
    - protocol: TCP
      port: 8080
```

**Test**:
```bash
# From frontend pod in production namespace
k exec -it deployment/frontend -n production -- curl http://backend-service:8080
# Should work ✓

# From any other pod
k exec -it deployment/untrusted -n production -- curl http://backend-service:8080
# Should timeout ✗
```

### Pattern 2: Allow from Different Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: frontend  # Allow from frontend namespace
    
    ports:
    - protocol: TCP
      port: 8080
```

**Setup**:
```bash
# Label the frontend namespace
k label namespace frontend environment=frontend

# Now frontend namespace pods can reach backend in production
```

### Pattern 3: Allow from POD in namespace + NAMESPACE selector combined

```yaml
ingress:
- from:
  # This is AND: must match BOTH conditions
  - podSelector:
      matchLabels:
        app: frontend      # Pod must have this label
    namespaceSelector:
      matchLabels:
        environment: prod  # AND Pod must be in namespace with this label
  
  ports:
  - protocol: TCP
    port: 8080
```

**Meaning**: "Allow pod with label app=frontend in namespace with label environment=prod"

### Pattern 4: Allow from Multiple Sources (OR logic)

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  
  - namespaceSelector:
      matchLabels:
        environment: external  # OR allow this namespace
  
  ports:
  - protocol: TCP
    port: 8080
```

**Meaning**: "Allow from frontend pods OR from external namespace" (either one).

### Pattern 5: Allow from Specific IP Block (External)

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 203.0.113.0/24      # External IP range
      except:
      - 203.0.113.10             # Except this IP
  
  ports:
  - protocol: TCP
    port: 8080
```

**Use case**: External monitoring service accessing your cluster.

### Pattern 6: Allow All Ingress (Empty From)

```yaml
ingress:
- from: []  # Empty = allow from ANYONE
  ports:
  - protocol: TCP
    port: 8080
```

**Meaning**: "Allow inbound on 8080 from anywhere" (basically no restriction).

---

## Common Ingress Mistakes

### Mistake 1: Forgot Port

```yaml
# ❌ WRONG - No port specified
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  # Missing ports! Does this allow port 8080? 443? ALL ports?

# ✅ CORRECT - Always specify port
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  ports:
  - protocol: TCP
    port: 8080
```

**Rule**: If you don't specify ports, traffic on ANY port is allowed (bad for security).

### Mistake 2: Forgot to Label Namespace

```yaml
# In network policy:
namespaceSelector:
  matchLabels:
    environment: prod

# But namespace isn't labeled!
k get ns prod --show-labels
# NAME   STATUS   AGE   LABELS
# prod   Active   5d    (none)

# Traffic still blocked!
# FIX:
k label namespace prod environment=prod
```

### Mistake 3: Mixing AND with OR incorrectly

```yaml
# ❌ WRONG - What does this mean?
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
    namespaceSelector:
      matchLabels:
        environment: prod
  - ipBlock:
      cidr: 10.0.0.0/8

# It's ambiguous. Does it mean:
# (frontend AND prod) OR 10.0.0.0/8 ?
# (frontend) AND (prod OR 10.0.0.0/8) ?

# ✅ CORRECT - Be explicit with multiple rules
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
    namespaceSelector:
      matchLabels:
        environment: prod
  ports:
  - protocol: TCP
    port: 8080

- from:
  - ipBlock:
      cidr: 10.0.0.0/8
  ports:
  - protocol: TCP
    port: 8080
```

---

# Part 3: Egress Rules (Outbound Traffic)

## Egress Rule Structure

```yaml
spec:
  podSelector:
    matchLabels:
      app: api
  
  policyTypes:
  - Egress
  
  egress:
  - to:
    - namespaceSelector: {}  # Allow to any namespace
    
    ports:
    - protocol: UDP
      port: 53               # DNS (CRITICAL!)
    - protocol: TCP
      port: 443              # HTTPS
```

**Meaning**: "Allow outbound to any namespace on UDP 53 (DNS) and TCP 443 (HTTPS)"

---

## Egress Patterns

### Pattern 1: Allow DNS Egress (ALWAYS NEEDED!)

```yaml
egress:
- to:
  - namespaceSelector: {}    # Anywhere
  ports:
  - protocol: UDP
    port: 53                 # DNS port

- to:
  - namespaceSelector: {}
  ports:
  - protocol: TCP
    port: 443                # HTTPS to external APIs
```

**Why DNS is critical**: If you block DNS, external service names won't resolve!

```bash
# Without DNS egress:
k exec <pod> -- curl https://example.com
# Hangs: can't resolve example.com

# With DNS egress (UDP 53):
k exec <pod> -- curl https://example.com
# Works: can resolve, then make HTTPS connection
```

### Pattern 2: Allow Egress to Internal Services

```yaml
egress:
- to:
  - podSelector:
      matchLabels:
        app: database
  ports:
  - protocol: TCP
    port: 5432  # Postgres
```

**Meaning**: "Allow outbound to database pods on port 5432"

### Pattern 3: Allow Egress to External API (with DNS!)

```yaml
egress:
# DNS MUST be first
- to:
  - namespaceSelector: {}
  ports:
  - protocol: UDP
    port: 53

# Then external API
- to:
  - ipBlock:
      cidr: 203.0.113.0/24   # External API IP range
  ports:
  - protocol: TCP
    port: 443
```

### Pattern 4: Deny Specific Egress

NetworkPolicy doesn't support explicit DENY (no -except- for egress).

**Workaround**: Use multiple Egress rules, one allows everything EXCEPT what you want blocked.

Actually, there's no clean way. Instead: Use what's allowed, don't mention what's blocked.

### Pattern 5: Egress to Any Internal Pod

```yaml
egress:
- to:
  - namespaceSelector: {}    # Any namespace
  ports:
  - protocol: TCP
    port: 443
```

---

## 2026 Exam Trap: Egress + DNS

**The most common failure**: Allow egress to external service but forget DNS.

```yaml
# ❌ WRONG - Pod can connect to external IP but can't resolve hostname
egress:
- to:
  - ipBlock:
      cidr: 203.0.113.0/24
  ports:
  - protocol: TCP
    port: 443

# Pod tries: curl https://api.example.com
# Hangs at DNS resolution (can't reach 8.8.8.8:53)
```

```yaml
# ✅ CORRECT - Allow DNS first, then external
egress:
# 1. DNS (always first!)
- to:
  - namespaceSelector: {}
  ports:
  - protocol: UDP
    port: 53

# 2. Then external API
- to:
  - ipBlock:
      cidr: 203.0.113.0/24
  ports:
  - protocol: TCP
    port: 443
```

**This was a REAL exam gotcha in 2026. Many candidates failed because Pod couldn't reach external APIs due to missing DNS egress.**

---

# Part 4: Default Deny Policies

## Strategy: Deny All, Then Allow What's Needed

This is "Zero Trust" networking.

### Step 1: Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}              # Apply to ALL pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny all
```

**Effect**: No pod can receive inbound traffic (unless other NP allows it).

### Step 2: Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all
```

**Effect**: Pods can't reach anything external.

### Step 3: Create Specific Allow Policies

```yaml
# Allow frontend -> backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
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

---

# Allow backend to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  
  # AND allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

**Full architecture**:
```
Frontend (port 80) ──[NP: frontend→backend:8080]──> Backend (port 8080)
                                                           |
                                                   [NP: backend→db:5432]
                                                           |
                                                      Database (port 5432)

All other traffic blocked (default deny)
```

---

# Part 5: Debugging NetworkPolicies

## Troubleshooting Workflow

### Issue: "Pod can't reach service"

```bash
# Step 1: Check if NetworkPolicy exists
k get networkpolicies -n production

# Step 2: Check what applies to the pod
k get networkpolicies -n production -o yaml | grep -A 20 "podSelector:"

# Step 3: Describe NetworkPolicy
k describe networkpolicy allow-frontend -n production

# Step 4: Test connectivity
k exec <source-pod> -- curl http://target-service:port
# If timeout: blocked by NetworkPolicy

# Step 5: Check labels match
k get pods --show-labels -n production
# Verify selectors in NP match actual pod labels

# Step 6: Check if DNS works
k exec <pod> -- nslookup example.com
# If fails: missing DNS egress rule
```

### Diagnosis Table

| Symptom | Likely Cause | Fix |
|---------|---|---|
| `curl target-service` hangs | No ingress rule on target | Add ingress rule |
| `curl target-service` works but `curl external.com` hangs | No egress DNS rule | Add UDP 53 egress |
| `curl external.com` still hangs after DNS | No egress HTTPS rule | Add TCP 443 egress |
| Pod selector doesn't match | Wrong labels | Fix pod labels or NP selector |
| Namespace selector doesn't work | Namespace not labeled | Label namespace |

---

# Part 6: Real Exam Scenarios

## Scenario 1: Implement Zero-Trust Network

**Question**: Secure production namespace with zero-trust networking. 
- Frontend can reach backend on 8080
- Backend can reach database on 5432 and external APIs on 443
- All else blocked

```bash
# 1. Create default-deny policies
k apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Allow from anything in same ns (exposed endpoint)
    ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
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
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  # Database
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # External APIs
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
EOF

# 2. Test
k exec deployment/frontend -n production -- curl http://backend-service:8080
# Should work ✓

k exec deployment/backend -n production -- curl https://api.example.com
# Should work (DNS + HTTPS allowed) ✓

k exec deployment/untrusted -n production -- curl http://backend-service:8080
# Should timeout ✗
```

## Scenario 2: Debug NetworkPolicy Blocking Traffic

**Question**: Backend pods can't reach external API. Debug and fix.

```bash
# 1. Check current policies
k get networkpolicies -n production
# See: allow-backend-egress

# 2. Describe it
k describe networkpolicy allow-backend-egress -n production
# Check: is DNS allowed? Is external IP allowed?

# 3. Test DNS
k exec deployment/backend -n production -- nslookup api.example.com
# Fails? Missing DNS egress

# 4. Fix: add DNS rule
k edit networkpolicy allow-backend-egress -n production
# Add:
# - to:
#   - namespaceSelector: {}
#   ports:
#   - protocol: UDP
#     port: 53

# 5. Retest
k exec deployment/backend -n production -- curl https://api.example.com
# Works ✓
```

## Scenario 3: Multi-Namespace Networking

**Question**: Allow traffic from frontend namespace to backend namespace.

```bash
# 1. Label namespaces
k label namespace frontend tier=presentation
k label namespace backend tier=application

# 2. Create policy in backend namespace
k apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-namespace
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: presentation
    ports:
    - protocol: TCP
      port: 8080
EOF

# 3. Test
k run test -n frontend --image=curlimages/curl -it -- curl http://api.backend:8080
# Should work ✓
```

---

# Part 7: Cheat Sheets

## NetworkPolicy Template (Copy-Paste Ready)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: myapp
  
  policyTypes:
  - Ingress
  - Egress
  
  # INGRESS RULES (inbound)
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  
  - from:
    - namespaceSelector:
        matchLabels:
          name: external
    ports:
    - protocol: TCP
      port: 443
  
  # EGRESS RULES (outbound)
  egress:
  # Allow DNS (ALWAYS INCLUDE THIS!)
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  
  # Allow to services in same namespace
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow to external API
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

## Quick Commands

```bash
# List all NetworkPolicies
k get networkpolicies -n <ns>

# Describe a policy
k describe networkpolicy <name> -n <ns>

# Create default deny
k apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Label namespace
k label namespace <ns> <key>=<value>

# Test connectivity
k exec <pod> -- curl http://target:port
k exec <pod> -- nslookup example.com

# Check pod labels
k get pods --show-labels -n <ns>
```

---

# Pre-Exam Checklist

- ✅ Understand: policyTypes behavior (Ingress/Egress/both)
- ✅ Understand: from/to selectors (pod, namespace, ipBlock)
- ✅ Know: DNS egress rule (UDP 53) is REQUIRED for external APIs
- ✅ Know: Empty from: [] means "allow from anywhere"
- ✅ Know: If egress specified, you MUST define rules (or default deny)
- ✅ Practice: Create default-deny + allow-specific policies <5 min
- ✅ Practice: Debug blocked traffic <3 min
- ✅ Practice: Multi-namespace policies <5 min
- ✅ Speed test: NetworkPolicy should take <10 min per question

---

# Common Exam Mistakes

❌ **Forgot DNS rule**  
→ External service names don't resolve  
→ FIX: Add `protocol: UDP, port: 53` egress

❌ **Added Egress but no rules**  
→ Pod can't reach ANYTHING  
→ FIX: Define at least DNS + your target egress rules

❌ **Wrong selector (typo in label)**  
→ Policy doesn't match pods  
→ FIX: Verify with `k get pods --show-labels`

❌ **Namespace not labeled**  
→ namespaceSelector doesn't work  
→ FIX: `k label namespace <ns> key=value`

❌ **Forgot port in ingress rule**  
→ ANY port allowed (defeats security)  
→ FIX: Always specify ports

