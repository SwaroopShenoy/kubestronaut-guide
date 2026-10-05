# Audit Logging

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 6 — Monitoring, Logging and Runtime Security (20%) · Prev: [Base images and Dockerfiles](../05-supply-chain-security/base-images-and-dockerfiles.md) · Next: [Falco](falco.md)

## Exam scope

**In scope:** writing an audit policy, enabling it on the API server, reading the log, and answering "who did what, when" with `jq`.

## How it works

The API server records each request as it passes through, according to an audit policy. Rules are evaluated top to bottom; the **first matching rule** decides the level.

Levels, least to most detail:

| Level | Records |
|---|---|
| `None` | Nothing |
| `Metadata` | User, verb, resource, timestamps, response code. No bodies |
| `Request` | Metadata plus the request body |
| `RequestResponse` | Metadata, request body and response body |

Stages: `RequestReceived`, `ResponseStarted`, `ResponseComplete`, `Panic`. Use `omitStages` to drop noisy ones (commonly `RequestReceived`).

## Step 1: Write the policy

Write it with `tee` (a plain `sudo cat > file` redirect runs as your user, not root):

```bash
sudo mkdir -p /etc/kubernetes/audit
sudo tee /etc/kubernetes/audit/policy.yaml >/dev/null <<'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
- RequestReceived
rules:
# Do not log the API server's own health checks or high-volume reads
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services"]
- level: None
  nonResourceURLs: ["/healthz*", "/readyz*", "/livez*"]

# Secret access: full detail for writes; metadata for reads
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]
- level: Metadata
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources: ["secrets"]

# Exec, attach and port-forward into pods (high-risk)
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach", "pods/portforward"]

# RBAC changes
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]

# Everything else
- level: Metadata
EOF
```

Notes on the policy:

- `pods/exec` is a subresource. In the log it appears as `objectRef.resource: "pods"` with `objectRef.subresource: "exec"`. Filter on both fields.
- The rule for secrets `get`/`list` is the one most default examples leave out. Without it, reads of credentials leave no trace beyond metadata.
- Keep `RequestResponse` narrow. It logs full bodies, which can contain the secret values themselves.

Validate the file as YAML before pointing the API server at it:

```bash
python3 -c 'import yaml,sys; yaml.safe_load(open("/etc/kubernetes/audit/policy.yaml")); print("ok")'
```

## Step 2: Enable on the API server

Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    volumeMounts:
    - name: audit-policy
      mountPath: /etc/kubernetes/audit
      readOnly: true
    - name: audit-log
      mountPath: /var/log/kubernetes/audit
  volumes:
  - name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
  - name: audit-log
    hostPath:
      path: /var/log/kubernetes/audit
      type: DirectoryOrCreate
```

The policy file and the log directory both need mounts. A missing mount is the most common reason the API server fails to start after this change.

The kubelet restarts the static pod when the manifest changes. Do not restart kubelet unless the pod does not come back.

```bash
kubectl get pods -n kube-system | grep kube-apiserver     # wait for Running
sudo tail -n 3 /var/log/kubernetes/audit/audit.log         # events should appear
```

If the API server does not return, the manifest has a problem. Use `sudo crictl ps -a | grep apiserver` and `sudo crictl logs <id>` to read the error. Fix the file; the kubelet will restart the pod.

## Step 3: Read the log

The log is **newline-delimited JSON**: one event per line. Parse each line, not the file as an array. Use `jq -c` on the stream, or `jq -s` to slurp it into an array.

```bash
LOG=/var/log/kubernetes/audit/audit.log

# Who created or changed secrets?
jq -c 'select(.objectRef.resource=="secrets" and (.verb=="create" or .verb=="update" or .verb=="patch" or .verb=="delete"))
       | {time: .requestReceivedTimestamp, user: .user.username, verb, name: .objectRef.name, ns: .objectRef.namespace, code: .responseStatus.code}' $LOG

# Shell access into pods
jq -c 'select(.objectRef.resource=="pods" and .objectRef.subresource=="exec")
       | {time: .requestReceivedTimestamp, user: .user.username, pod: .objectRef.name, ns: .objectRef.namespace}' $LOG

# Failed requests
jq -c 'select(.responseStatus.code >= 400) | {time: .requestReceivedTimestamp, user: .user.username, verb, resource: .objectRef.resource, code: .responseStatus.code}' $LOG

# RBAC changes
jq -c 'select(.objectRef.resource | test("role|rolebinding"; "i")) | {time: .requestReceivedTimestamp, user: .user.username, verb, name: .objectRef.name}' $LOG
```

Useful fields:

| Field | Meaning |
|---|---|
| `requestReceivedTimestamp` | When the request arrived (use this for "when") |
| `user.username` | Who (ServiceAccounts appear as `system:serviceaccount:<ns>:<name>`) |
| `verb` | `get`, `list`, `create`, `update`, `patch`, `delete`, `watch` |
| `objectRef.resource` / `.subresource` / `.namespace` / `.name` | What |
| `responseStatus.code` | Result (`401`/`403` = denied) |
| `sourceIPs` | Client address |

## Step 4: Answer the investigation question

Example: "Who created secret `db-password` in production?"

```bash
jq -c 'select(.objectRef.resource=="secrets" and .objectRef.name=="db-password" and .verb=="create")
       | {time: .requestReceivedTimestamp, user: .user.username, code: .responseStatus.code}' "$LOG"
```

Read the `user.username` field. If it is a ServiceAccount, trace it to the workload that holds that token.

## Common mistakes

- Using `.[]` on the log. It is not a JSON array.
- Filtering exec with `.objectRef.resource=="pods/exec"`. The resource is `pods`; `exec` is the subresource.
- Forgetting the log volume mount, so the file is written inside the container and lost.
- Setting `RequestResponse` on everything. The log fills and the disk fills with it.
- Writing the policy with `sudo cat > file`. The redirect runs without sudo.

## Practice

1. Enable the policy; run `kubectl create secret generic demo --from-literal=a=b`; find the event with `jq`.
2. Run `kubectl exec` into a pod; find the `pods` / `exec` event and the user.
3. Create a failing request as an unprivileged user; filter for 403s.

## Quick reference

```bash
sudo tail -f /var/log/kubernetes/audit/audit.log | jq -c .
jq -c 'select(.objectRef.subresource=="exec")' /var/log/kubernetes/audit/audit.log
jq -s 'length' /var/log/kubernetes/audit/audit.log      # count events
```
