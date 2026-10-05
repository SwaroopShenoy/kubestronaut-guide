# Threat Model and Attack Paths

Up: [KCSA hub](../../KCSA_Crash_Course.md) · Domain 4 — Kubernetes Threat Model (16%) · Prev: [Secrets](../03-kubernetes-security-fundamentals/secrets.md) · Next: [Supply chain and images](../05-platform-security/supply-chain-and-images.md)

## STRIDE

A way to ask what can go wrong for each component or data flow.

| Threat | Meaning | Kubernetes example |
|---|---|---|
| Spoofing | Pretending to be someone else | Stolen ServiceAccount token; forged client certificate |
| Tampering | Changing data or code | Modified image; altered manifest; edited etcd data |
| Repudiation | Denying an action | No audit log; logs that can be deleted |
| Information disclosure | Exposing data | Readable Secrets; open kubelet API; exposed dashboard |
| Denial of service | Making a service unavailable | Resource exhaustion; API flooding; no limits set |
| Elevation of privilege | Gaining more access | Privileged pod; `create pods` used to mount the host; RBAC escalation |

Map each threat to a control. Repudiation maps to audit logs; elevation maps to RBAC, PSA and admission.

## Trust boundaries

- Untrusted: internet users, external clients.
- Edge: ingress controllers and load balancers.
- Workloads: application pods.
- Control plane: API server, etcd, controllers. Most trusted.
- Nodes: run workloads and hold kubelet credentials.

Each boundary crossing needs authentication and a control: TLS on ingress, RBAC on API calls, NetworkPolicy between pods, encryption for etcd.

## Common attack paths

1. **Container escape to node**: a vulnerable app gives code execution; a privileged pod or kernel exploit leaves the container; the attacker reads node credentials.
2. **RBAC escalation**: a ServiceAccount with `create pods` launches a pod with a host mount or a privileged profile, then reads node or other ServiceAccount tokens.
3. **Exposed API or kubelet**: anonymous access to the API or the kubelet read or exec API.
4. **Supply chain**: a malicious or vulnerable base image or dependency runs in the cluster.
5. **Cloud metadata**: a pod reaches the instance metadata endpoint and obtains node credentials.
6. **Lateral movement**: no NetworkPolicy; a compromised pod reaches databases and other namespaces.

## Attacker techniques to recognize (MITRE ATT&CK for containers)

- Initial access: exposed API, compromised credentials, vulnerable app, malicious image.
- Execution: exec into a container, a malicious workload.
- Persistence: a rogue CronJob, modified RBAC, malicious admission webhook.
- Credential access: ServiceAccount token theft, reading Secrets, metadata access.
- Discovery: `kubectl auth can-i --list`, DNS enumeration.
- Defense evasion: disabling logs, deleting evidence.

## Incident response

Phases: preparation, detection, containment, eradication, recovery, lessons learned.

Containment actions in Kubernetes:

- Isolate the pod with a deny-all NetworkPolicy label.
- Revoke or rotate the ServiceAccount token and the credentials it used.
- Cordon and drain a compromised node.
- Block egress to known command-and-control addresses.

Forensics sources: API audit log, container logs, runtime alerts, network flow records, and the node itself if it can be preserved.

## Practice questions

- Which STRIDE category does an unlogged change belong to? (Repudiation)
- A pod with `create pods` mounts the node filesystem. Which threat is this? (Elevation of privilege)
- What is the first containment step for a suspicious pod? (Isolate it, for example with a deny-all NetworkPolicy, while preserving evidence)
- Which boundary is most trusted? (The control plane)
