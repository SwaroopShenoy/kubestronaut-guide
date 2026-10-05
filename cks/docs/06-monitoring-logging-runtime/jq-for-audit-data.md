# jq for Audit and JSON Data

Up: [CKS hub](../../README.md) · Domain 6 — Monitoring, Logging and Runtime Security (20%) · Prev: [Falco](falco.md) · Next: [Rego basics](../reference/rego-basics.md)

Audit logs are structured JSON, and jq is the fastest way to ask them questions. This topic covers the filters and transforms that answer the common ones.

jq is a tool, not a domain. It is in this guide because audit logs are JSON and the exam expects quick answers from them.

## Input shape matters

| Input | Use |
|---|---|
| One JSON object per line (audit log, many API tools) | `jq -c 'filter' file` processes each line |
| One JSON document (`kubectl get ... -o json`) | `jq '.items[] | ...'` |
| Need to group or count across lines | add `-s` (slurp) to make one array |

The audit log is line-delimited. `jq '.[]' audit.log` fails because there is no outer array.

## Basics

```bash
jq '.'                              # pretty print
jq -c '.'                           # one line per object
jq -r '.user.username'              # raw string, no quotes
jq '.items[] | .metadata.name'      # from kubectl output
```

## Filtering

```bash
jq -c 'select(.verb == "delete")'
jq -c 'select(.objectRef.resource == "secrets" and .verb == "create")'
jq -c 'select(.responseStatus.code >= 400)'
jq -c 'select(.user.username | startswith("system:serviceaccount:"))'
jq -c 'select(.user.username | test("admin"; "i"))'
```

`and`, `or`, and `not` are jq keywords. Parentheses group.

## Shaping output

```bash
jq -c '{time: .requestReceivedTimestamp, user: .user.username, verb, resource: .objectRef.resource}'
jq -r '[.requestReceivedTimestamp, .user.username, .verb, .objectRef.resource] | @csv'
jq -r '[.user.username, .verb] | @tsv'
```

`{verb}` is shorthand for `{verb: .verb}`.

## Counting and grouping (needs slurp)

```bash
jq -s 'length' audit.log                                         # number of events
jq -s 'group_by(.user.username) | map({user: .[0].user.username, n: length}) | sort_by(-.n)' audit.log
jq -s '[.[] | .user.username] | unique' audit.log
```

## Working with kubectl output

```bash
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.securityContext.runAsNonRoot != true) | "\(.metadata.namespace)/\(.metadata.name)"'
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'
```

## Missing fields

Accessing a missing field returns `null`, not an error. Use `//` for defaults and `?` to suppress errors on mismatched types:

```bash
jq -r '.objectRef.subresource // "-"' audit.log
jq -r '.. | .image? // empty' pod.json          # every image string anywhere in the document
```

## Common mistakes

- Using `.[]` on line-delimited input.
- Forgetting `-c` and getting a wall of indented output when you wanted one line per event.
- Comparing numbers as strings. `"403"` and `403` are different; use the number from `responseStatus.code`.
- Filtering exec on `objectRef.resource == "pods/exec"`. Use `subresource`.

## Quick reference

```bash
jq -c 'select(...)' file
jq -s 'length' file
jq -r '[fields] | @csv' file
```
