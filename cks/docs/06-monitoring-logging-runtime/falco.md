# Falco (Runtime Threat Detection)

Up: [CKS hub](../../README.md) · Domain 6 — Monitoring, Logging and Runtime Security (20%) · Prev: [Audit logging](audit-logging.md) · Next: [Container immutability at runtime](runtime-immutability.md)

Some attacks happen inside a running container, where no scanner looks. This topic covers Falco, which watches system calls and raises alerts, and how to read what it reports and adjust how it behaves.

## Exam scope

**In scope:** confirming Falco is running and producing alerts, and reading an alert. The Falco documentation (falco.org/docs) is on the exam's allowed-resources list, so Falco is used on the exam.

**Unclear:** how far tasks go in editing rules. The curriculum says "detect" and does not mention rule authoring. Reading and modifying an existing rule (changing a condition, an output or a priority, or adding an exception) is plausible and is covered below. Writing a large rule set from scratch is not described anywhere in the curriculum.

The details below were checked against Falco 0.45.0 and its default rule files. Other versions differ in places, so confirm against the version installed on the cluster you work with. Install Falco from the official chart or package at falco.org.

## What Falco does

Falco watches system calls (kernel events, collected by an eBPF probe or a kernel module) and matches them against rules. A match produces an alert with a priority, a message, and fields such as the container, the process and the user.

The collection method is the `engine` setting in `falco.yaml`. Current versions default to `modern_ebpf`; `kmod` (kernel module) is the alternative.

## Verify it is running

```bash
kubectl get pods -n falco -o wide           # one pod per node when deployed as DaemonSet
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20
```

On a package install:

```bash
sudo systemctl status falco
sudo journalctl -u falco -f
```

The label and namespace depend on how Falco was installed. If the commands above find nothing, list the pods with `kubectl get pods -A | grep -i falco`.

## Trigger and read an alert

Two classic triggers:

```bash
kubectl exec -it deploy/app -n production -- sh            # shell with a terminal in a container
kubectl exec deploy/app -n production -- cat /etc/shadow    # sensitive file read
```

Then read the alert:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco | grep -i "shell"
```

An alert from the default "Terminal shell in container" rule looks like this:

```
12:04:31.123456789: Notice A shell was spawned in a container with an attached terminal | evt_type=execve user=root user_uid=0 process=sh proc_exepath=/bin/sh parent=runc command=sh terminal=34816 ...
```

The values above are illustrative. Read an alert in this order: the priority (`Notice`), the message (what happened), then the fields that identify the process. With the default setting `append_output` with `suggested_output: true`, Falco also appends fields such as the container id, container name, image, and for Kubernetes workloads the namespace and pod name, so the alert tells you which workload it came from.

## Where Falco reads its configuration and rules

The main file is `/etc/falco/falco.yaml`. These settings matter most:

```yaml
rules_files:
- /etc/falco/falco_rules.yaml
- /etc/falco/falco_rules.local.yaml
- /etc/falco/rules.d
```

| File or directory | Purpose |
|---|---|
| `falco_rules.yaml` | The default (stable) rules that ship with Falco; do not edit |
| `falco_rules.local.yaml` | Local overrides to the default rules |
| `rules.d/` | Additional rule files; files are loaded from this directory |

Files are loaded in order, so a later file can override something defined in an earlier one.

Other settings:

```yaml
json_output: false        # true prints each alert as one JSON object per line
priority: debug           # minimum priority that is reported
stdout_output:
  enabled: true           # on by default
file_output:
  enabled: false
```

Setting `json_output: true` makes alerts easy to filter with `jq` (see [jq for audit data](jq-for-audit-data.md)).

Any setting can also be overridden on the command line with `-o key=value`, for example `falco -o json_output=true`.

## Default rule tiers

The Falco rules project publishes three sets of rules:

| Set | Contents | Installed by default |
|---|---|---|
| Stable | Well-tested rules such as "Terminal shell in container", "Read sensitive file untrusted", "Drop and execute new binary in container", "Linux Kernel Module Injection Detected" | Yes |
| Incubating | Rules still maturing, such as "Launch Package Management Process in Container" | No |
| Sandbox | Experimental rules, such as the "Write below etc" and "Write below binary dir" family | No |

So whether a given rule exists on a cluster depends on which sets were installed. Check what is actually loaded instead of assuming:

```bash
sudo grep -rh '^- rule:' /etc/falco/ | sort
```

## Rule structure

Three building blocks:

- `rule`: name, `desc`, `condition`, `output`, `priority`, optional `tags`
- `macro`: a reusable condition fragment
- `list`: a reusable set of values

```yaml
# /etc/falco/rules.d/local.yaml
- list: admin_shells
  items:
  - bash
  - sh

- macro: production_container
  condition: container.name startswith "prod-"

- rule: Shell in production container
  desc: An interactive shell was started in a production container
  condition: >
    spawned_process and container and production_container
    and proc.name in (admin_shells)
  output: >
    Shell in production (user=%user.name container=%container.name proc=%proc.name cmd=%proc.cmdline)
  priority: WARNING
  tags:
  - shell
  - production
```

Output fields use `%` syntax: `%user.name`, `%container.name`, `%proc.cmdline`, `%fd.name`.

Priority levels, from most to least severe: `EMERGENCY`, `ALERT`, `CRITICAL`, `ERROR`, `WARNING`, `NOTICE`, `INFORMATIONAL`, `DEBUG`.

## Common condition building blocks

```text
spawned_process          a process was started
container                event is inside a container
shell_procs              a shell binary (macro in the default rules)
proc.name                executable name
proc.cmdline             full command line
user.name                user running the event
fd.name                  file path involved in an open or write
fd.sip / fd.dip          source / destination IP
fd.dport                 destination port
evt.type                 syscall type (open, write, connect, ...)
```

The full list of fields available in the installed version is printed by `falco --list`.

## Changing a default rule

Do not edit `falco_rules.yaml`; upgrades overwrite it. Put changes in `falco_rules.local.yaml` or `rules.d/`, using `override`. The `override` section says, for each key, whether the new value is appended to the original or replaces it.

Narrow a rule's condition (append):

```yaml
- rule: Terminal shell in container
  condition: and not container.image.repository = "registry.example.com/debug-tools"
  override:
    condition: append
```

Change the priority of a rule (replace):

```yaml
- rule: Terminal shell in container
  priority: WARNING
  override:
    priority: replace
```

Switch a rule off (replace):

```yaml
- rule: Terminal shell in container
  enabled: false
  override:
    enabled: replace
```

Add to a list or macro:

```yaml
- list: admin_shells
  items:
  - zsh
  override:
    items: append
```

`append` and `replace` cannot be mixed within one override block.

## Exceptions

An exception describes cases that should not trigger the rule, without rewriting the condition. Each exception has a `name`, the `fields` to compare, the comparison operators in `comps`, and the `values` to match:

```yaml
- rule: Write below binary dir
  exceptions:
  - name: proc_writer
    fields:
    - proc.name
    - fd.directory
    comps:
    - "="
    - "="
    values:
    - - my-installer
      - /usr/local/bin
```

To add values to an exception that already exists in a default rule:

```yaml
- rule: Write below binary dir
  exceptions:
  - name: proc_writer
    values:
    - - apk
      - /usr/lib/alpine
  override:
    exceptions: append
```

## Load and test a rule

Validate the files before restarting. A syntax error in a rules file stops Falco from starting:

```bash
sudo falco -V /etc/falco/rules.d/local.yaml
sudo falco -V /etc/falco/falco_rules.yaml -V /etc/falco/rules.d/local.yaml
```

Reload after changes:

```bash
sudo systemctl restart falco              # package install
kubectl rollout restart ds/falco -n falco # DaemonSet install
```

Then trigger the behaviour the rule describes and read the output. A rule that validates can still never match, for example when a field name is wrong.

## Outputs

Alerts go to standard output by default, which is the container log in a DaemonSet and the journal in a package install. Other destinations are configured in `falco.yaml`:

```yaml
http_output:
  enabled: true
  url: https://alerts.example.com/falco
file_output:
  enabled: true
  filename: /var/log/falco/events.log
```

Falco can also write to syslog or run a program for each alert. Falcosidekick is a separate forwarder that sends alerts to many other systems.

Do not hardcode webhook URLs in shared files; use a secret-managed value.

## Common mistakes

- Writing a rule that never matches because a field name is wrong for the installed Falco version. Test with a trigger, not by reading the rule.
- Forgetting to restart or reload after editing a rules file.
- Putting custom rules in the default rules file. Edits there are overwritten on upgrade. Use `rules.d/` or `falco_rules.local.yaml`.
- Alerting on every shell in every namespace. Scope with a macro or container label.
- Assuming a rule from the documentation exists on the cluster. Rules in the incubating and sandbox sets are not installed by default.
- Using `append` for a key where the documentation says `replace`, or both in one override.

## Quick reference

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=100
sudo journalctl -u falco -f
ls /etc/falco/rules.d/
sudo falco -V <rules-file>
sudo falco --list
```

---

Prev: [Audit logging](audit-logging.md) · Next: [Container immutability at runtime](runtime-immutability.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
