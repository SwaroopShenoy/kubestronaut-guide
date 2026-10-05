# Runtime Security

Up: [KCSA hub](../../KCSA_Crash_Course.md) · Domain 5 — Platform Security (16%) · Prev: [Admission and policy](admission-and-policy.md) · Next: [CIS benchmark and audit](../06-compliance-and-frameworks/cis-benchmark-and-audit.md)

## Prevention versus detection

- Prevention (scanning, admission, RBAC, SecurityContext) stops known bad states before they run.
- Runtime security detects what a running workload actually does. It catches zero-days and misuse that prevention allowed.

## Falco

Falco is a CNCF graduated runtime threat-detection project. It watches system calls (through eBPF or a kernel module) and applies rules to flag suspicious behavior.

Typical detections:

- A shell started inside a container.
- Reads of sensitive files such as `/etc/shadow`.
- Writes to system directories.
- Unexpected outbound network connections.
- Privilege escalation attempts.

A rule has a condition (what to match), an output message, and a priority. Teams add their own rules for their applications.

## Other runtime tools

- Tetragon: eBPF-based enforcement and observability (Cilium project).
- Commercial platforms built on similar ideas.

## Scanners and runtime tools are different

| Tool | Question it answers |
|---|---|
| Trivy (scanner) | Does this image contain known vulnerabilities? (before running) |
| Falco (runtime) | Is this running workload doing something suspicious? (while running) |
| Audit log | What requests did the API server receive? |
