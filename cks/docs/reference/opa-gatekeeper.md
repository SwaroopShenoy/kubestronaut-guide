# OPA Gatekeeper

Up: [CKS hub](../../README.md) · Reference · Related: [Admission control](../05-supply-chain-security/admission-control.md) · Prev: [Rego basics](rego-basics.md)

## Scope

Gatekeeper is not named in the official CKS curriculum. Treat it as useful background for admission control; confirm against the official curriculum before investing study time in it.

## Model

- **ConstraintTemplate** — defines a new kind (the CRD) and the Rego that produces violations.
- **Constraint** — an instance of that kind: what it matches, and its parameters.

Install from the release manifest that matches your Gatekeeper version (see open-policy-agent.github.io/gatekeeper). Do not copy an unversioned `master` URL into production docs.

```bash
kubectl get pods -n gatekeeper-system
```

## Example: required labels

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
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("missing required labels: %v", [missing])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-owner
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
  parameters:
    labels: ["owner"]
```

Test:

```bash
kubectl create namespace test-no-owner     # rejected
kubectl create namespace test-owner --dry-run=server -o yaml   # add label owner=team-a to pass
```

## enforcementAction

| Value | Behavior |
|---|---|
| `deny` | Reject the request (default) |
| `dryrun` | Record violations only; nothing is blocked |
| `warn` | Allow, and return a warning to the client |

`audit` is not a valid value. Use `dryrun` to test a new constraint against existing objects; violations appear in the Constraint's status:

```bash
kubectl get k8srequiredlabels must-have-owner -o jsonpath='{.status.violations}{"\n"}'
```

## Using the community library

The Gatekeeper library (github.com/open-policy-agent/gatekeeper-library) provides tested templates for common checks: allowed repositories, privileged containers, required probes, host paths, and more. Using a library template is faster and less error-prone than writing Rego from scratch. Read its parameters before applying.

## Debugging

```bash
kubectl get constrainttemplates
kubectl get constraints                     # all constraint kinds
kubectl describe k8srequiredlabels must-have-owner
kubectl logs -n gatekeeper-system -l control-plane=controller-manager --tail=50
```

Common failures:

- Template applied but its CRD does not exist yet. Wait for the template to report status `created`.
- Constraint matches nothing. Check the `match.kinds` apiGroup: Deployments are `apps`, Pods are `""`.
- Policy blocks system namespaces. Add `excludedNamespaces` (for example `kube-system`).

## Common mistakes

- Matching Deployments and writing the Rego against `spec.containers` instead of `spec.template.spec.containers`.
- Forgetting `excludedNamespaces`, which can block the cluster's own components.
- Shipping `deny` before running `dryrun` against existing workloads.

---

Prev: [Rego basics](rego-basics.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
