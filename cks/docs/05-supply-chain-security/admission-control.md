# Admission Control

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 5 — Supply Chain Security (20%) · Prev: [Image signing with Cosign](image-signing-cosign.md) · Next: [Base images and Dockerfiles](base-images-and-dockerfiles.md)

Admission control runs after authentication and authorization, before an object is persisted. It is where policies that reject bad pods (unsigned images, root containers, forbidden registries) actually get enforced.

## Options, in order of how often you will meet them

| Mechanism | Built-in | Language | Use |
|---|---|---|---|
| Pod Security Admission | Yes | Labels | Pod-level profiles per namespace; see [Security context and PSS](../04-microservice-vulnerabilities/security-context-and-pss.md) |
| ValidatingAdmissionPolicy | Yes (GA 1.30) | CEL | Custom validation with no webhook to run |
| ImagePolicyWebhook | Yes (plugin) | External webhook | Image trust decisions from an external service |
| OPA Gatekeeper / Kyverno | No (install) | Rego / YAML | Full policy frameworks; see [OPA Gatekeeper](../reference/opa-gatekeeper.md) |

## ValidatingAdmissionPolicy (CEL, built in)

Require non-root pods in a namespace:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-nonroot
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: >-
      has(object.spec.securityContext) &&
      has(object.spec.securityContext.runAsNonRoot) &&
      object.spec.securityContext.runAsNonRoot == true
    message: "pods must set spec.securityContext.runAsNonRoot: true"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-nonroot-prod
spec:
  policyName: require-nonroot
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: production
```

Test:

```bash
kubectl apply -f root-pod.yaml      # expect: denied with the message above
```

This policy rejects a pod that sets `runAsNonRoot` only on the container. Adjust the expression if you need container-level checks; the pod-level field is what the sample checks.

## ImagePolicyWebhook (native plugin)

The API server calls an external service that answers allow or deny for each image. Configuration is a file passed to the API server, which contains the webhook kubeconfig and timing.

```yaml
# /etc/kubernetes/admission/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/admission/webhook-kubeconfig.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false    # fail closed
```

Enable it in the kube-apiserver manifest:

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/admission/admission-config.yaml
```

Mount both files into the static pod (hostPath volumes), as with the encryption config.

Pitfall: `defaultAllow: true` means "allow when the webhook is unreachable". That is fail-open and defeats the purpose. Use `false` in any security context.

## OPA Gatekeeper and Kyverno

These are installed add-ons. The exam is more likely to test reading and adjusting a given policy than writing a new framework from scratch. See:

- [OPA Gatekeeper](../reference/opa-gatekeeper.md) for ConstraintTemplate and Constraint structure
- [Rego basics](../reference/rego-basics.md) for reading Rego

## Verifying signed images at admission

Gatekeeper cannot verify image signatures by itself. Signature verification at admission is done by a verifier such as Kyverno's `verifyImages` rule, sigstore's policy-controller, or the ImagePolicyWebhook backend. See [Image signing with Cosign](image-signing-cosign.md) for the signing side.

## Common mistakes

- Using `failurePolicy: Ignore` on a security policy. A broken webhook then allows everything.
- Creating a policy with `validationActions: [Audit]` and believing it blocks. Audit only records.
- Deploying a policy before testing it. Use `--dry-run=server` or warn/audit modes first.
- Writing a Gatekeeper constraint with `enforcementAction: audit`. The valid values are `deny`, `dryrun`, and `warn`.

## Quick reference

```bash
kubectl get validatingadmissionpolicy,validatingadmissionpolicybinding
kubectl apply --dry-run=server -f <manifest>
kubectl get pods -n kube-system | grep kube-apiserver
```
