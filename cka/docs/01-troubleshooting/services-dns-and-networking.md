# Services, DNS and Networking Troubleshooting

Up: [CKA hub](../../README.md) · Domain 1 — Troubleshooting (30%) · Prev: [Control plane and nodes](control-plane-and-nodes.md) · Next: [RBAC](../02-cluster-architecture-installation-config/rbac.md)

A Service that routes nowhere looks healthy from every dashboard. This topic shows how to find the broken link between a name, a Service, its endpoints and the network path behind them.

## Service has no endpoints

The most common networking failure. A Service with no endpoints sends traffic nowhere.

```bash
kubectl get svc <svc> -o wide
kubectl get endpoints <svc>             # or: kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl describe svc <svc> | grep -i selector
kubectl get pods -l <selector-key>=<selector-value> --show-labels
```

Diagnosis:

- Selector does not match pod labels → fix the label on the pod template or the selector on the Service.
- Pods match but are not Ready → readiness probe failing; check `describe pod`.
- `targetPort` does not match the container's listening port → fix `targetPort`. Note that an endpoint can exist with a bad port; the Service will still fail.

## Testing connectivity from inside the cluster

Use a throwaway pod:

```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh
# inside:
nslookup <svc>
wget -qO- -T 3 http://<svc>:<port>
```

For a specific pod IP, test the pod directly to separate Service problems from pod problems:

```bash
kubectl get pod <pod> -o wide
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://<pod-ip>:<port>
```

## DNS

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
kubectl get svc -n kube-system kube-dns
cat /etc/resolv.conf               # inside a pod
```

Expected name format: `<svc>.<namespace>.svc.cluster.local`. A pod that cannot resolve `kubernetes.default` usually has CoreDNS down or a broken `kube-dns` Service.

If CoreDNS pods are crashing, check the Corefile ConfigMap:

```bash
kubectl -n kube-system get configmap coredns -o yaml
```

## kube-proxy

kube-proxy programs iptables or IPVS rules that make Service IPs work.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50
```

Symptom: a Service works from some nodes but not others → check kube-proxy on the failing node.

## CNI

Pod-to-pod traffic depends on the network plugin.

```bash
ls /etc/cni/net.d/                 # on the node
kubectl get pods -n kube-system -o wide | grep -iE 'calico|flannel|cilium|weave'
kubectl describe node <node> | grep -i networkunavailable
```

If a node reports `NetworkUnavailable`, the CNI pod on that node is the first thing to check.

## NetworkPolicy blocking traffic

```bash
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>
```

A common trap: a new policy with `policyTypes: [Egress]` and no DNS rule. Symptom: `nslookup` times out while direct IP access works. See [NetworkPolicy](../03-services-networking/network-policy.md).

## Quick reference

```bash
kubectl get endpoints <svc>
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- nslookup <svc>
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl get netpol -A
```
