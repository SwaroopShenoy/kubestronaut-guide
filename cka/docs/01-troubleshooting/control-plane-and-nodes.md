# Control Plane and Nodes

Up: [CKA hub](../../CKA_2026_Complete_Crash_Course.md) · Domain 1 — Troubleshooting (30%) · Prev: [Troubleshooting method](troubleshooting-method.md) · Next: [Services, DNS and networking](services-dns-and-networking.md)

## Model

Control-plane components (kube-apiserver, kube-controller-manager, kube-scheduler, etcd) run as **static pods**. The kubelet on the control-plane node reads manifests from `/etc/kubernetes/manifests/` and runs them. The kubelet itself is a systemd service, so a broken kubelet means nothing runs on that node.

| Component | Manifest | Kubeconfig |
|---|---|---|
| kube-apiserver | `/etc/kubernetes/manifests/kube-apiserver.yaml` | none (uses certs directly) |
| kube-controller-manager | `/etc/kubernetes/manifests/kube-controller-manager.yaml` | `/etc/kubernetes/controller-manager.conf` |
| kube-scheduler | `/etc/kubernetes/manifests/kube-scheduler.yaml` | `/etc/kubernetes/scheduler.conf` |
| etcd | `/etc/kubernetes/manifests/etcd.yaml` | none (uses certs directly) |
| kubelet | systemd unit, drop-ins in `/etc/systemd/system/kubelet.service.d/` | `/etc/kubernetes/kubelet.conf` |

Kubelet configuration file: `/var/lib/kubelet/config.yaml` (confirm with `ps -ef | grep kubelet`).

## Triage: is the API up?

```bash
# From a workstation or the control-plane node
kubectl get --raw /readyz?verbose | tail -5
kubectl get pods -n kube-system -o wide
```

If the API is down, use the node directly:

```bash
sudo crictl ps -a | grep -E 'kube-apiserver|etcd'
sudo crictl logs <container-id>
sudo journalctl -u kubelet --no-pager -n 100
```

`crictl` works without the API server, so it is the reliable tool here.

## Static pod manifest errors

Symptom: a control-plane component is missing from `kubectl get pods -n kube-system`, or the API server disappears after an edit.

```bash
sudo journalctl -u kubelet | grep -iE 'error|failed' | tail -20
sudo crictl ps -a | grep kube-apiserver
```

Common causes:

- YAML syntax error (indentation, a stray character)
- Wrong flag name
- Wrong file path in a volume mount or flag (certificate not at the path you gave)
- Wrong image tag

Fix the manifest in place. The kubelet notices the change and restarts the pod, usually within about 30 seconds. Do not move the manifest out of the directory to "restart" it unless you need to stop the component; if you do, move it back.

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
watch -n2 'sudo crictl ps | grep kube-apiserver'
```

## Scheduler and controller manager

Symptoms: pods stuck `Pending` with no scheduling events (scheduler), or Deployments that never create ReplicaSets (controller manager).

```bash
kubectl get pods -n kube-system -l component=kube-scheduler
kubectl logs -n kube-system kube-scheduler-<node>
grep -E 'kubeconfig|server' /etc/kubernetes/scheduler.conf
ls -l /etc/kubernetes/scheduler.conf /etc/kubernetes/controller-manager.conf
```

A wrong `server:` address in the kubeconfig is the usual cause. The kubeconfig should point at the API server address reachable from the control-plane node.

## Node NotReady

```bash
kubectl get nodes
kubectl describe node <node> | sed -n '/Conditions:/,/Addresses:/p'
```

Read the condition that is `False` or `Unknown`:

| Condition | Typical cause | Check on the node |
|---|---|---|
| Ready=False / Unknown | kubelet stopped, certificate expired, network down | `systemctl status kubelet`, `journalctl -u kubelet` |
| DiskPressure | Images or logs filling disk | `df -h`, `sudo crictl rmi --prune` |
| MemoryPressure | Pods over limits, host process | `free -h`, `kubectl top pods -A` |
| PIDPressure | Runaway processes | `ps aux \| wc -l`, `sudo crictl ps \| wc -l` |
| NetworkUnavailable | CNI not running | `ls /etc/cni/net.d/`, CNI pods in `kube-system` |

Then on the node:

```bash
sudo systemctl status kubelet --no-pager
sudo journalctl -u kubelet -n 100 --no-pager
sudo systemctl restart kubelet         # after fixing the cause, not as the fix itself
```

## Kubelet will not start

**Swap is on.** Kubelet refuses to run with swap enabled by default.

```bash
free -h | grep -i swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab     # make it permanent
```

**Wrong config path or bad YAML.**

```bash
sudo journalctl -u kubelet | grep -iE 'config|yaml' | tail
sudo cat /var/lib/kubelet/config.yaml
sudo grep -n -- '--config' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

**Wrong API server address in kubelet.conf.**

```bash
sudo grep 'server:' /etc/kubernetes/kubelet.conf
# If it points at 127.0.0.1 on a worker, change it to the control-plane address
sudo sed -i 's|https://127.0.0.1:6443|https://10.0.0.5:6443|' /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

**Container runtime not running.** The kubelet cannot start containers without it.

```bash
sudo systemctl status containerd --no-pager
sudo crictl info
```

## Drain and uncordon

```bash
kubectl cordon <node>                                        # no new pods
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>                                      # make schedulable again
```

A node left cordoned after a fix is a common exam miss. Always uncordon as the last step.

## Certificates that expired

Symptom: x509 errors in kubelet or API server logs.

```bash
sudo kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate
```

Renew with kubeadm, then restart the control-plane components:

```bash
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

Renewing `all` also rewrites kubeconfig files. Copy the new admin kubeconfig to `~/.kube/config` if your workstation uses it.

## Practice

1. Break `kube-scheduler.yaml` by changing one flag; confirm the scheduler disappears; restore it.
2. Set `--anonymous-auth` to an invalid value in the API server manifest; diagnose from `crictl logs`; fix it.
3. Stop kubelet on a worker, make the node NotReady, bring it back, uncordon it.
4. Add swap back in a lab node and confirm kubelet refuses to start.

## Quick reference

```bash
sudo crictl ps -a
sudo crictl logs <id>
sudo journalctl -u kubelet -n 100 --no-pager
kubectl get --raw /readyz?verbose
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data && kubectl uncordon <node>
```
