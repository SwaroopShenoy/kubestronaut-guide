# Image Signing & Cosign Masterclass
## CKS Deep Dive - Supply Chain Security & Signature Verification

---

# Part 1: The Problem (Why Image Signing Matters)

## Attack Scenario: Unsigned Image

```
Attacker compromises container registry
    ↓
Modifies image (injects malware)
    ↓
Your cluster pulls "trusted" image
    ↓
Runs malicious code (you have no way to know)
    ↓
Full cluster compromise
```

**Default Docker/Kubernetes**: No signature verification. Trusts ANY image.

**Solution**: Sign images at build time, verify signatures at runtime.

---

# Part 2: Image Signing Concepts

## What is an Image Signature?

**Image Signature** = Cryptographic proof that image hasn't been tampered with.

**Components**:
1. **Image Digest** (SHA-256 hash of image contents)
2. **Private Key** (signer's secret - used to create signature)
3. **Signature** (proof that private key holder created this image)
4. **Public Key** (verification - anyone can check signature)

**Flow**:
```
Build time:
Image → (hash) → Digest
Image + Private Key → (sign) → Signature
Store in registry: Image + Signature

Verification time:
Fetch Image + Signature from registry
Fetch Public Key
Verify: Signature matches Image digest?
    YES → Pull and run image
    NO → Reject (image tampered!)
```

---

# Part 3: Cosign Basics

## What is Cosign?

**Cosign** = Container Signing and Verification tool (sigstore project).

**Simple flow**:
```bash
# At build time
cosign sign --key cosign.key gcr.io/myapp:v1

# At runtime (in Kubernetes)
cosign verify --key cosign.pub gcr.io/myapp:v1
```

---

## Step 1: Generate Signing Keys

```bash
# Generate key pair
cosign generate-key-pair

# Creates:
# - cosign.key (PRIVATE - keep secret!)
# - cosign.pub (PUBLIC - share freely)

# Secure the private key
chmod 600 cosign.key
# Store in CI/CD secret (GitHub Actions, GitLab CI, etc.)
```

**Alternative: Use OIDC (no key management)**:
```bash
# GitHub Actions example (automatic OIDC token)
cosign sign --oidc-issuer-url=https://token.actions.githubusercontent.com \
  --identity-token=$ACTIONS_ID_TOKEN \
  gcr.io/myapp:v1
# No need to manage private keys!
```

---

## Step 2: Sign Image at Build Time

### In Docker Build Pipeline

```bash
# After building and pushing image
docker build -t gcr.io/myapp:v1 .
docker push gcr.io/myapp:v1

# Sign the image
cosign sign --key cosign.key gcr.io/myapp:v1

# Output: Signature stored in registry (as separate artifact)
```

### In CI/CD (GitHub Actions Example)

```yaml
# .github/workflows/build.yml
name: Build and Sign
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # Required for OIDC
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Build image
      run: |
        docker build -t gcr.io/${{ github.repository }}:${{ github.sha }} .
        docker push gcr.io/${{ github.repository }}:${{ github.sha }}
    
    - name: Install cosign
      uses: sigstore/cosign-installer@v3
    
    - name: Sign image (OIDC - no key management!)
      run: |
        cosign sign --oidc-issuer-url=https://token.actions.githubusercontent.com \
          --identity-token=${{ secrets.GITHUB_TOKEN }} \
          gcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## Step 3: Verify Signature Manually

```bash
# Install cosign
wget https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign-linux-amd64
chmod +x cosign-linux-amd64

# Verify signature (requires public key)
./cosign-linux-amd64 verify --key cosign.pub \
  gcr.io/myapp:v1

# Output:
# Verification successful!
# [{
#   "critical": {...},
#   "optional": {...}
# }]

# If signature invalid:
# Error: no valid signatures found
```

---

# Part 4: Enforce Signature Verification in Kubernetes

## Option 1: Admission Controller (Simple)

Use Kyverno or Kubewarden policy to enforce signed images.

### Kyverno Policy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  validationFailureAction: enforce  # Block unsigned images
  rules:
  - name: check-signature
    match:
      resources:
        kinds:
        - Pod
    verifyImages:
    - imageReferences:
      - "gcr.io/*"
      attestors:
      - name: "check-cosign-signature"
        entries:
        - keys:
            publicKeys: |
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
            signatureAlgorithm: sha256
```

**How it works**:
- User tries to deploy pod with image `gcr.io/myapp:v1`
- Kyverno intercepts (validates webhook)
- Fetches image signature from registry
- Verifies signature with public key
- If valid: Pod deploys ✅
- If invalid: Pod rejected ❌

---

## Option 2: OPA/Gatekeeper Policy

```rego
# verify-image-signature.rego
package k8simagesignatures

violation[{"msg": msg}] {
  # Only check gcr.io images (others are exempt)
  container := input.review.object.spec.containers[_]
  image := container.image
  startswith(image, "gcr.io/")
  
  # Check if image has valid signature
  # (In real exam, you'd integrate with cosign verify API)
  not image_has_valid_signature[image]
  
  msg := sprintf("Image %v not signed", [image])
}

# Mock: In real world, fetch from registry and verify
image_has_valid_signature[img] {
  # Would call: cosign verify --key cosign.pub <img>
  img == "gcr.io/trusted-image:v1"
}
```

---

## Option 3: Manual Verification in Deployment

```bash
# Before deploying, manually verify image
IMAGE=gcr.io/myapp:v1

cosign verify --key cosign.pub $IMAGE
# If error, stop deployment

# Then apply manifests
k apply -f deployment.yaml
```

---

# Part 5: Real Exam Scenarios

## Scenario 1: Sign Image and Verify Signature

**Question**: Sign an image with cosign, then verify it works.

```bash
# 1. Generate keys (first time only)
cosign generate-key-pair
# Creates: cosign.key, cosign.pub

# 2. Build and push image
docker build -t myregistry/myapp:v1 .
docker push myregistry/myapp:v1

# 3. Sign the image
cosign sign --key cosign.key myregistry/myapp:v1

# 4. Verify signature
cosign verify --key cosign.pub myregistry/myapp:v1
# Output: Verification successful!

# 5. Try to verify tampered image (should fail)
# (Attacker modifies image in registry)
cosign verify --key cosign.pub myregistry/myapp:v1
# Output: Error: no valid signatures found
```

## Scenario 2: Enforce Signatures with Kyverno

**Question**: Deploy Kyverno and enforce image signatures. Block unsigned images.

```bash
# 1. Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace

# 2. Create policy
k apply -f - <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-signatures
spec:
  validationFailureAction: enforce
  rules:
  - name: check-signature
    match:
      resources:
        kinds:
        - Pod
    verifyImages:
    - imageReferences:
      - "myregistry/*"
      attestors:
      - name: cosign-check
        entries:
        - keys:
            publicKeys: |
              -----BEGIN PUBLIC KEY-----
              ... YOUR cosign.pub CONTENT ...
              -----END PUBLIC KEY-----
            signatureAlgorithm: sha256
EOF

# 3. Test: Try to deploy unsigned image (should fail)
k run unsigned-pod --image=myregistry/unsigned:v1
# Error: Policy verification failed

# 4. Test: Deploy signed image (should succeed)
k run signed-pod --image=myregistry/signed:v1
# Pod created
```

## Scenario 3: Cosign with OIDC (No Key Management)

**Question**: Set up cosign signing in CI/CD without managing private keys.

```bash
# In CI/CD (GitHub Actions)
cat <<'EOF' > .github/workflows/sign.yml
name: Sign Image
on: [push]

jobs:
  sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # Critical: OIDC token
    
    steps:
    - name: Install cosign
      uses: sigstore/cosign-installer@v3
    
    - name: Sign image (OIDC)
      env:
        REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
        REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
      run: |
        # Login to registry
        echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USERNAME" --password-stdin
        
        # Push image
        docker build -t myregistry/app:${{ github.sha }} .
        docker push myregistry/app:${{ github.sha }}
        
        # Sign with OIDC (no private key needed!)
        cosign sign --oidc-issuer-url=https://token.actions.githubusercontent.com \
          --identity-token=${{ secrets.GITHUB_TOKEN }} \
          myregistry/app:${{ github.sha }}
EOF
```

**Benefits**:
- ✅ No private key to manage (GitHub holds OIDC token)
- ✅ Each build gets unique identity
- ✅ Easy to rotate (automatic)
- ✅ Compliance-friendly (audit trail in GitHub)

---

# Part 6: Debugging Signature Issues

## Signature Verification Fails

```bash
# Issue 1: Public key mismatch
cosign verify --key wrong-key.pub myregistry/app:v1
# Error: no valid signatures found
# FIX: Use correct cosign.pub that matches the private key used to sign

# Issue 2: Image not actually signed
cosign verify --key cosign.pub myregistry/unsigned:v1
# Error: no valid signatures found
# FIX: Sign it first with cosign sign --key cosign.key myregistry/unsigned:v1

# Issue 3: Signature exists but corrupted
# (Attacker tries to tamper with signature artifact)
cosign verify --key cosign.pub myregistry/tampered:v1
# Error: invalid signature
# FIX: Re-sign the image from trusted build pipeline
```

## Kyverno Policy Not Enforcing

```bash
# Check 1: Is Kyverno running?
k get pod -n kyverno
# Should show kyverno-* pods running

# Check 2: Is policy loaded?
k get clusterpolicy verify-signatures
# Should show policy

# Check 3: Is validationFailureAction set to enforce?
k describe clusterpolicy verify-signatures | grep validationFailureAction
# Should show: enforce (not audit)

# Check 4: Check webhook logs
k logs -n kyverno deployment/kyverno | grep -i "verify\|image\|signature"

# FIX: Policy syntax error?
k apply -f policy.yaml
# Should not show validation error
```

---

# Part 7: Best Practices

## 1. Sign ALL Images

✅ **RIGHT**:
```bash
# Sign every image at build time
cosign sign --key cosign.key myregistry/app:v1
```

❌ **WRONG**:
```bash
# Only sign "important" images
# (Attacker uses unsigned image)
```

## 2. Store Public Keys Safely

✅ **RIGHT**:
```bash
# Public key in Git (it's public!)
# Or in Kyverno policy (embedded)
```

❌ **WRONG**:
```bash
# Keep public key secret (defeats purpose)
# Store in private repo (hard to distribute)
```

## 3. Use OIDC for CI/CD (Not Local Keys)

✅ **RIGHT**:
```bash
cosign sign --oidc-issuer-url=... --identity-token=$TOKEN
```

❌ **WRONG**:
```bash
# Store private key in CI/CD secret
# (If CI/CD compromised, all images can be re-signed)
```

## 4. Enforce Signatures at Admission (Webhook)

✅ **RIGHT**:
```yaml
validationFailureAction: enforce  # Block unsigned images
```

❌ **WRONG**:
```yaml
validationFailureAction: audit  # Only log, don't block
# (Attacker can still deploy unsigned images)
```

---

# Part 8: Cheat Sheet

## Quick Sign

```bash
# Generate keys (one time)
cosign generate-key-pair

# Sign image (after push)
cosign sign --key cosign.key myregistry/app:v1
```

## Quick Verify

```bash
# Verify manually
cosign verify --key cosign.pub myregistry/app:v1

# In Kyverno policy
verifyImages:
  - imageReferences:
    - "myregistry/*"
    attestors:
    - name: cosign-check
      entries:
      - keys:
          publicKeys: |
            -----BEGIN PUBLIC KEY-----
            ... cosign.pub content ...
            -----END PUBLIC KEY-----
```

## Quick OIDC (No Keys)

```yaml
# GitHub Actions
env:
  COSIGN_EXPERIMENTAL: 1

run: |
  cosign sign --oidc-issuer-url=https://token.actions.githubusercontent.com \
    --identity-token=${{ secrets.GITHUB_TOKEN }} \
    myregistry/app:${{ github.sha }}
```

---

# Pre-Exam Checklist

- ✅ Understand: Image signature = cryptographic proof image not tampered
- ✅ Generate cosign key pair (cosign generate-key-pair)
- ✅ Sign image after push (cosign sign --key cosign.key <image>)
- ✅ Verify signature works (cosign verify --key cosign.pub <image>)
- ✅ Understand OIDC signing (no local key management)
- ✅ Deploy Kyverno with image verification policy
- ✅ Test: Unsigned image rejected, signed image accepted
- ✅ Understand: Block at admission time (webhook)
- ✅ Speed: Should take <10 min to sign + deploy verification policy

---

# Speed Targets for Exam

- **Generate cosign keys**: <1 min
- **Sign image**: <2 min
- **Verify signature manually**: <1 min
- **Deploy Kyverno policy**: <5 min
- **Test enforcement (unsigned rejected)**: <3 min
- **Debug signature failures**: <5 min

---

# Important Note for CKS Exam

**You will NOT write Cosign code from scratch.**

**What you WILL do**:
- ✅ Understand image signatures (what they are, why they matter)
- ✅ Run `cosign sign` and `cosign verify` commands
- ✅ Deploy Kyverno/admission policy to enforce signatures
- ✅ Debug why signatures fail

**What you probably WON'T do**:
- ❌ Implement OIDC token exchange logic
- ❌ Debug cryptography internals
- ❌ Build custom signature formats

**Focus**: Practical usage, enforcement, troubleshooting.

