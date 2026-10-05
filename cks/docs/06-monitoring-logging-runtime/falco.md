# Falco (Runtime Threat Detection)

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 6 — Monitoring, Logging and Runtime Security (20%) · Prev: [Audit logging](audit-logging.md) · Next: [jq for audit data](jq-for-audit-data.md)

## Exam scope

**In scope:** confirming Falco is running and producing alerts, reading an alert, understanding rule structure, and writing or editing a rule to match a given behavior.

The earlier course suggested installing Falco with `apt-key` and a daemonset URL. Both are outdated. Use the official chart or package from falco.org and follow its install docs for the installed version.

## What Falco does

Falco watches system calls (kernel events, via eBPF or a kernel module) and matches them against rules. A match produces an alert with a priority, the rule name, and fields such as container name and process.

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

## Trigger and read an alert

Safe, classic triggers:

```bash
kubectl exec -it deploy/app -n production -- sh          # shell spawned in a container
kubectl exec deploy/app -n production -- cat /etc/shadow  # sensitive file read, depending on rules
```

Then read the alert:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco | grep -i "shell"
```

Alerts look like:

```
Warning A shell was spawned in a container (user=root container=app-7d... shell=sh ...)
```

Read: priority, rule (the human sentence), then the fields that identify the workload.

## Rule structure

Rules live in the rules files and in `/etc/falco/rules.d/` for local additions. Three building blocks:

- `rule` — name, `desc`, `condition`, `output`, `priority`, optional `tags`
- `macro` — reusable condition fragment
- `list` — reusable set of values

```yaml
# /etc/falco/rules.d/local.yaml
- list: admin_shells
  items: [bash, sh]

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
  tags: [shell, production]
```

The default rule set already includes a shell-in-container rule using macros such as `spawned_process`, `container`, and `shell_procs`. Before writing a new rule, check whether a default covers it; duplicate alerts are noise.

Output fields use `%` syntax: `%user.name`, `%container.name`, `%proc.cmdline`, `%fd.name`.

## Common condition building blocks

```text
spawned_process          a process was started
container                event is inside a container
proc.name                executable name
proc.cmdline             full command line
user.name                user running the event
fd.name                  file path involved in an open/write
fd.sip / fd.dip          source / destination IP
fd.dport                 destination port
evt.type                 syscall type (open, write, connect, ...)
```

Field and macro names vary between Falco versions. Confirm names with the installed version's docs or by reading the default rules file that ships with it before relying on them.

## Load and test a rule

Reload after changes:

```bash
sudo systemctl restart falco              # package install
kubectl rollout restart ds/falco -n falco # DaemonSet install
```

Validate the rules file before reloading. Falco has a rule-validation mode; check `falco --help` for the flag your version supports.

## Outputs

Alerts go to stdout/journal by default. Route them to HTTP in `falco.yaml`:

```yaml
http_output:
  enabled: true
  url: https://alerts.example.com/falco
```

Do not hardcode webhook URLs in shared files; use a secret-managed value.

## Common mistakes

- Writing a rule that never matches because a field name is wrong for the installed Falco version. Test with a trigger, not by reading the rule.
- Forgetting to restart or reload after editing a rules file.
- Putting custom rules in the default rules file. Edits there are overwritten on upgrade. Use `rules.d/`.
- Alerting on every shell in every namespace. Scope with a macro or container label.

## Practice

1. Confirm Falco is running on every node.
2. Spawn a shell in a test pod; find the alert.
3. Write a rule that alerts when `/etc/passwd` is opened for writing inside a container, trigger it, and read the output.

## Quick reference

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=100
sudo journalctl -u falco -f
ls /etc/falco/rules.d/
```
