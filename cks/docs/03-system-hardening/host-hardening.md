# Host (Node) Hardening

Up: [CKS hub](../../README.md) · Domain 3 — System Hardening (10%) · Prev: [Cluster upgrades](../02-cluster-hardening/cluster-upgrades.md) · Next: [AppArmor and seccomp](apparmor-and-seccomp.md)

This doc is **new**. The original course had only SSH and kernel module snippets; this fills in the host-footprint and access items.

## Exam scope

**In scope:** reducing what runs on a node, restricting who can log in, restricting which ports are reachable, blocking unneeded kernel modules, and keeping packages patched.

## 1. Reduce the attack surface

List what is listening and what starts at boot:

```bash
sudo ss -tlnp                                  # listening TCP sockets with process names
systemctl list-unit-files --state=enabled       # services that start at boot
```

Expected on a worker: container runtime, kubelet, kube-proxy (or CNI agent), SSH only if required. Anything else should be disabled:

```bash
sudo systemctl disable --now <unit>
```

Remove packages the node does not need:

```bash
apt list --installed | grep -Ei "telnet|rsh|nis|talk|avahi|cups"
sudo apt-get purge -y <package>
```

## 2. SSH access

```bash
sudo vi /etc/ssh/sshd_config
# PermitRootLogin no
# PasswordAuthentication no
# PubkeyAuthentication yes
sudo sshd -t && sudo systemctl reload ssh
```

Always run `sshd -t` before reloading, and keep an existing session open while testing a new one. A bad config can lock you out.

## 3. Kernel modules

Block modules that are not used by the cluster. Install directives map the module to a no-op:

```bash
echo "install dccp /bin/true" | sudo tee /etc/modprobe.d/disable-dccp.conf
echo "install sctp /bin/true" | sudo tee /etc/modprobe.d/disable-sctp.conf    # only if your CNI does not need SCTP
sudo modprobe -r dccp 2>/dev/null
lsmod | grep -E "^dccp" || echo "dccp not loaded"
```

Do not blacklist modules a CNI or storage driver needs (for example `overlay` and `br_netfilter` for container networking). Check before disabling.

## 4. Firewall the node

Limit the kubelet API (10250) and etcd (2379–2380) to the networks that need them:

```bash
sudo ufw allow from 10.0.0.0/8 to any port 10250 proto tcp
sudo ufw allow from 10.0.0.5 to any port 2379 proto tcp     # control-plane IP only
sudo ufw default deny incoming
```

Keep SSH reachable before enabling a default-deny policy.

## 5. Patching

```bash
sudo apt-get update
sudo apt-get -s upgrade | grep -i security        # simulate, show security updates
sudo apt-get install -y unattended-upgrades
```

Pin the Kubernetes packages separately (see [cluster upgrades](../02-cluster-hardening/cluster-upgrades.md)) so an OS upgrade does not change the kubelet version.

## 6. Cloud IAM for nodes

Nodes should get the smallest IAM role that lets them register and pull images. Anything broader is a hardening finding. The exam is Kubernetes-focused, so this is a concept check rather than a console task.

## 7. Check the work

```bash
sudo ss -tlnp
systemctl list-unit-files --state=enabled | grep -Ev "kubelet|containerd|crio|ssh"
lsmod | grep -E "dccp|sctp"
```

## Common mistakes

- Blocking SSH on a remote node and losing access.
- Blacklisting a kernel module the CNI needs; pods then fail to get network.
- Applying a default-deny firewall before adding the rule for the SSH session in use.
- Editing `sshd_config` without `sshd -t`.

## Quick reference

```bash
sudo ss -tlnp
systemctl list-unit-files --state=enabled
sudo sshd -t
sudo modprobe -r <module>
```
