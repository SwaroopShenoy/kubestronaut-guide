# Rego Basics

Up: [CKS hub](../../README.md) · Reference · Prev: [jq for audit data](../06-monitoring-logging-runtime/jq-for-audit-data.md) · Next: [OPA Gatekeeper](opa-gatekeeper.md)

## Scope

Rego is the policy language used by OPA and Gatekeeper. The scope is:

- **Reading** a Rego rule and understanding what it rejects: expected.
- **Writing or editing** a policy, including a ConstraintTemplate: not expected. The examples here are reading material. Confirm with the official curriculum.

## Core model

A Rego file defines rules. The Gatekeeper rule that matters is `violation`. Each time `violation` is satisfied, the object is rejected with the message.

```rego
package k8srequirednonroot

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("container %v must set runAsNonRoot", [container.name])
}
```

Read it as: "for each container, if runAsNonRoot is not set, produce this message."

## Syntax rules that trip people up

- **Statements inside a rule body are AND-ed.** Put them on separate lines (or separate with `;`). There is no `AND` keyword.
- **OR is multiple rules.** Two `violation` rules with different conditions mean "either violates".
- **`[_]` iterates.** `containers[_]` yields each element in turn.
- **`not x` is true when `x` is undefined or false.** A missing field makes `not` succeed.
- **`:=` assigns; `==` compares; `=` unifies.** Use `:=` for new variables.
- **Membership:** `x in [a, b]` or `some x in list`. `in` does not take parenthesized tuples.
- **Strings:** `startswith(s, prefix)`, `endswith`, `contains`, `sprintf("%v", [x])`.
- **Counting:** `count(array)`, `sort(array)`. There is no pipe operator.

## Iterating and filtering

```rego
# Any container image not from an allowed registry
violation[{"msg": msg}] {
  some container in input.review.object.spec.containers
  not allowed_image(container.image)
  msg := sprintf("image %v is not from an allowed registry", [container.image])
}

allowed_image(img) {
  some registry in input.parameters.allowedRegistries
  startswith(img, registry)
}
```

`input.parameters` comes from the Constraint's `spec.parameters`.

## Input shape in Gatekeeper

```text
input.review.object         the Kubernetes object being admitted
input.review.operation      CREATE, UPDATE, DELETE
input.review.kind           group/version/kind
input.review.namespace      namespace of the request
input.parameters            values from the Constraint
```

Pod-controller objects (Deployment, StatefulSet) put the pod under `spec.template.spec`. A rule written for Pods that never checks the template will silently ignore Deployments.

```rego
pod_spec := input.review.object.spec.template.spec
```

Use that when the constraint matches controllers as well as Pods.

## Testing

Use the Rego playground or `opa eval`:

```bash
opa eval -d policy.rego -i input.json "data.k8srequirednonroot.violation"
```

Build the input JSON from a real object: `kubectl get pod x -o json` gives `object` directly; wrap it as `{"review": {"object": ...}}`.

## Common mistakes

- Writing `container.resources.limits` alone, which is truthy only when it is set; use `not` to test absence.
- Using `AND` or `OR` keywords (not valid Rego).
- Checking only `spec.containers` and forgetting `initContainers` and `ephemeralContainers`.
- Writing a rule that compiles but matches nothing because the path is one level off.
- Using `(a, b)` for membership; use `[a, b]`.

## Quick reference

```text
violation[{"msg": msg}] { ... }     define a violation
x := value                          assign
x == value                          compare
not expr                            absence / negation
some item in list                   iterate
startswith(s, p)                    string prefix
count(list)                         length
```
