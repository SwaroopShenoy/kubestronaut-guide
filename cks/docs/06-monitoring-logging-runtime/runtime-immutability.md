# Container Immutability at Runtime

Up: [CKS README](../../README.md) · Domain 6 — Monitoring, Logging and Runtime Security (20%) · Prev: [Falco](falco.md) · Next: [jq for audit data](jq-for-audit-data.md)

A container that cannot change after it starts is much harder to turn into an attack platform. This topic covers how to make containers immutable and how to notice when something tries to change one.

Official curriculum topic: "Ensure immutability of containers at runtime".

## The idea

Anything an attacker needs to do after gaining a foothold, such as writing a tool to disk, installing a package or editing a configuration file, requires a writable filesystem and a shell. Remove those and the options shrink.

## Making a container immutable

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: immutable-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: registry.example.com/team/app@sha256:<digest>
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: config
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: tmp
    emptyDir: {}
  - name: config
    configMap:
      name: app-config
```

What each setting contributes:

| Setting | Effect |
|---|---|
| `readOnlyRootFilesystem: true` | The container's filesystem cannot be written; only listed volumes are writable |
| `emptyDir` on a known path | Gives the application a scratch area without opening the root filesystem |
| `readOnly: true` on mounts | Configuration cannot be altered from inside |
| `allowPrivilegeEscalation: false` | Blocks setuid tricks that gain new privileges |
| `capabilities.drop: ALL` | Removes powers such as changing file ownership or loading modules |
| Image by digest | The running image cannot be swapped by moving a tag |
| Minimal or distroless image | No shell or package manager to abuse |

Do not set a writable `hostPath` mount; it defeats the isolation.

## Checking it works

```bash
kubectl exec immutable-app -- touch /usr/test        # expect: Read-only file system
kubectl exec immutable-app -- touch /tmp/test        # succeeds, scratch volume
kubectl exec immutable-app -- sh                     # expect: no shell in a distroless image
```

## Controlling who can change a running container

- Restrict `create` on `pods/exec`, `pods/attach` and `pods/ephemeralcontainers` with RBAC. Ephemeral containers let someone attach a new container to a running pod.
- Audit these calls (see [Audit logging](audit-logging.md)).
- Use admission policy so that pods without `readOnlyRootFilesystem: true` are rejected where that is the standard.

## Detecting changes

Falco watches system calls and can alert when something modifies a container that should be immutable. Which rules exist depends on the rule sets installed (see the rule tiers in [Falco](falco.md)). In Falco 0.45.0:

- The **stable** set, installed by default, includes "Drop and execute new binary in container" (an executable that was not part of the image is run), "Terminal shell in container", "Execution from /dev/shm", "Fileless execution via memfd_create", and "Read sensitive file untrusted".
- The **incubating** set includes "Launch Package Management Process in Container".
- The **sandbox** set includes the "Write below etc", "Write below binary dir" and "Modify binary dirs" rules.

Incubating and sandbox rules are not loaded unless they were added, so confirm what is running before relying on one:

```bash
sudo grep -rh '^- rule:' /etc/falco/ | sort
```

See [Falco](falco.md) for how to read an alert, where rules live, and how to adjust them.

## Immutable infrastructure

The same principle applies a level up: replace pods and nodes rather than patching them in place. Change the image, roll out a new Deployment revision, and discard the old pods.

## Common mistakes

- Setting `readOnlyRootFilesystem: true` and forgetting a writable path the application needs, so it crashes at startup.
- Using a mutable tag such as `latest` for a container that is supposed to be fixed.
- Locking down the container but leaving `pods/exec` open to every user.

---

Prev: [Falco](falco.md) · Next: [jq for audit data](jq-for-audit-data.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
