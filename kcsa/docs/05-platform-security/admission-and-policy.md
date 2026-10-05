# Admission and Policy

Up: [KCSA hub](../../README.md) · Domain 5 — Platform Security (16%) · Prev: [Supply chain and images](supply-chain-and-images.md) · Next: [Runtime security](runtime-security.md)

Admission is where a cluster decides whether a change is acceptable. This chapter covers the built-in admission controls and the policy engines that extend them.

## Where policy runs

After authentication and authorization, the API server runs admission. Admission can:

- **Mutate** a request (add defaults, inject sidecars or labels), then
- **Validate** it (accept or reject).

Mutating webhooks run before validating ones.

## Built-in options

- **Pod Security Admission**: enforces Pod Security Standards per namespace.
- **ResourceQuota and LimitRanger**: limit resource use and set defaults.
- **NodeRestriction**: limits what each kubelet can modify.
- **ValidatingAdmissionPolicy**: custom validation rules written in CEL, evaluated without a webhook.
- **ImagePolicyWebhook**: asks an external service whether an image is allowed.

## Policy engines

| Engine | Language | Notes |
|---|---|---|
| OPA Gatekeeper | Rego | Constraint templates and constraints; Open Policy Agent is a CNCF project |
| Kyverno | YAML | Validate, mutate and generate resources; CNCF project |

Policy engines let you write rules such as "every pod must set `runAsNonRoot`" or "images must come from our registry". Start in audit or warn mode to see impact before enforcing.

## Fail open vs fail closed

If a webhook is unreachable, `failurePolicy: Fail` rejects requests (fail closed) and `Ignore` allows them (fail open). For security policy, fail closed is the safer default, with the tradeoff that a broken webhook can block deployments.
