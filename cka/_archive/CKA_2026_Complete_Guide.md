# CKA 2026 Complete Crash Course
## Raw Knowledge You Need to Know

---

## 🔐 Certificates & PKI - Deep Dive

**Where certs live**:
```bash
/etc/kubernetes/pki/
  ├── ca.crt / ca.key              # Cluster CA
  ├── apiserver.crt / apiserver.key
  ├── apiserver-kubelet-client.crt / .key
  ├── front-proxy-ca.crt / .key
  ├── front-proxy-client.crt / .key
  └── etcd/
      ├── ca.crt / ca.key
      ├── server.crt / server.key
      ├── peer.crt / peer.key
      └── healthcheck-client.crt / .key
```

**Check cert expiry**:
```bash
# Single cert
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates -subject

# All certs at once
kubeadm certs check-expiration

# Manual loop
for cert in /etc/kubernetes/pki/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in "$cert" -noout -dates -subject 2>/dev/null
done

# Check etcd certs
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -dates
openssl x509 -in /etc/kubernetes/pki/etcd/peer.crt -noout -dates
```

**Certificate fields to understand**:
```bash
# Check CN (Common Name), O (Organization), SANs (Subject Alt Names)
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A 5 "Subject:\|DNS:"

# Example output:
# Subject: CN = kube-apiserver
# DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, 
# DNS:kubernetes.default.svc.cluster.local, DNS:<CONTROL-PLANE-IP>
```

**Why certs fail**:
- Expired (check dates!)
- Wrong issuer (must be signed by CA)
- Subject/SAN mismatch (API server MUST have correct DNS names)
- Permission denied on key file

**Renew certs**:
```bash
# Renew all
kubeadm certs renew all

# Renew specific
kubeadm certs renew apiserver
kubeadm certs renew kubelet-client-current  # For kubelet client cert

# CRITICAL: Restart control plane after renew
sudo systemctl daemon-reload
sudo systemctl restart kubelet
# OR move static pod manifests to trigger restart:
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 10
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

**kubelet certificates on worker nodes**:
```bash
# Kubelet client cert (used to talk to API server)
ls -la /var/lib/kubelet/pki/

# Check kubelet.conf (kubeconfig for kubelet)
cat /etc/kubernetes/kubelet.conf

# If kubelet cert expired:
# 1. Check kubelet logs
journalctl -u kubelet -f

# 2. Check config
cat /var/lib/kubelet/config.yaml

# 3. Renew from control plane (kubelet-client cert)
kubeadm certs renew kubelet-client-current

# 4. Restart kubelet
sudo systemctl restart kubelet
```

**CSR (Certificate Signing Requests)**:
```bash
# List pending CSRs
k get csr

# Approve CSR
k certificate approve <csr-name>

# Deny CSR
k certificate deny <csr-name>

# Check CSR details
k describe csr <csr-name>
k get csr <csr-name> -o yaml
```

---

## 🔧 Kubelet Troubleshooting - The Deep Dive

**Kubelet is the key to node problems. Master this.**

### Kubelet Basics
- Runs on every node (including control plane)
- Manages Pods via API server
- Watches `/etc/kubernetes/manifests/` for static pods
- Uses kubeconfig at `/etc/kubernetes/kubelet.conf`

### Kubelet Status Check
```bash
# From control plane - check node
k get nodes
k describe node <node-name>

# On the node itself
systemctl status kubelet
systemctl is-enabled kubelet  # Should be enabled

# View logs (real-time)
journalctl -u kubelet -f

# View last N lines
journalctl -u kubelet -n 50 --no-pager

# View since X ago
journalctl -u kubelet --since "30m ago"

# Search for errors
journalctl -u kubelet | grep ERROR
journalctl -u kubelet | grep "kubeconfig"
```

### Common Kubelet Issues

**1. Node NotReady**
```bash
# From control plane
k describe node <node> 
# Look at "Conditions" section - what's NotReady?

# On the node
systemctl status kubelet
journalctl -u kubelet -f

# Common causes:
# - kubelet service stopped: systemctl start kubelet
# - swap enabled: swapoff -a
# - cgroup issues: check cgroup driver mismatch
# - disk pressure: df -h
# - memory pressure: free -h
# - PID pressure: ps aux | wc -l
```

**2. Kubelet can't start**
```bash
# Check config syntax
cat /var/lib/kubelet/config.yaml
# Look for YAML errors

# Check kubeconfig
cat /etc/kubernetes/kubelet.conf
# Verify paths exist:
#   server: https://CONTROL-PLANE-IP:6443
#   certificate-authority-data: (valid base64)
#   client-certificate-data: (valid base64)
#   client-key-data: (valid base64)

# Common: kubelet.conf has wrong server IP
# Fix: Update server field and restart
sudo sed -i 's/old-ip/new-ip/g' /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

**3. Kubelet can't reach API server**
```bash
# Logs show: connection refused, x509 cert errors
journalctl -u kubelet | grep -i "certificate\|connection refused"

# Solutions:
# 1. Check API server is running
systemctl status kubelet  # On control plane
k get pod -n kube-system -l component=kube-apiserver

# 2. Check kubelet.conf has correct server
cat /etc/kubernetes/kubelet.conf | grep server:

# 3. Check certs are valid
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# 4. Verify network connectivity
curl -k --cacert /var/lib/kubelet/pki/kubelet-client-current.pem \
  --cert /var/lib/kubelet/pki/kubelet-client-current.pem \
  https://CONTROL-IP:6443/api/v1
```

**4. Static pods not starting**
```bash
# Kubelet watches /etc/kubernetes/manifests/ for static pods
ls -la /etc/kubernetes/manifests/

# If pod manifest has error:
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Fix the YAML, save, kubelet restarts pod in ~30s
watch k get pods -n kube-system | grep apiserver

# Check kubelet logs for parse errors
journalctl -u kubelet | grep "failed to parse\|error parsing"
```

**5. Swap is enabled (kubelet refuses)**
```bash
# Kubelet REFUSES to start if swap is on
free -h | grep Swap

# Disable swap
sudo swapoff -a

# Permanently (comment out in /etc/fstab)
sudo vi /etc/fstab
# Comment out swap line

# Then restart
sudo systemctl start kubelet
```

**6. Kubelet config path wrong**
```bash
# Kubelet started but can't read config
journalctl -u kubelet | grep "error loading config\|no such file"

# Find where config SHOULD be
ps aux | grep kubelet | grep -o "\-\-config=/[^ ]*"
# Output: --config=/var/lib/kubelet/config.yaml

# If file doesn't exist or permission denied
ls -la /var/lib/kubelet/config.yaml
sudo cat /var/lib/kubelet/config.yaml

# Fix path in systemd service
sudo vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Check: --config=/var/lib/kubelet/config.yaml

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Debug Kubelet via crictl
```bash
# crictl = container runtime CLI (works even if kubelet broken)

# List all containers
crictl ps -a

# Check kubelet container (if running as container)
crictl ps | grep kubelet

# Check kubelet logs via runtime
crictl logs <kubelet-container-id>

# Check static pod containers
crictl ps -a | grep kube-apiserver

# Get logs of static pod
crictl logs <container-id>

# Stop/start container directly (dangerous!)
crictl stop <container-id>
crictl start <container-id>
```

---

## 📚 kubectl api-resources & API Discovery

**You MUST know this command - exams test it.**

```bash
# List ALL available resource types
k api-resources

# Output columns:
# NAME (e.g., pods, services)
# SHORTNAMES (e.g., po, svc)
# APIVERSION (e.g., v1, apps/v1)
# NAMESPACED (true/false)
# KIND

# Examples from output:
# pods          po       v1           true      Pod
# services      svc      v1           true      Service
# deployments   deploy   apps/v1      true      Deployment
# nodes         no       v1           false     Node
# clusterroles  cr       rbac.authorization.k8s.io/v1  false  ClusterRole
```

**Why it matters**:
- You don't know if a resource is namespaced? Use api-resources!
- Is it `Certificate` or `Cert`? Check!
- What version should I use? Check!

**Finding specific resources**:
```bash
# Search for cert-related resources
k api-resources | grep -i cert

# Output might be:
# certificatesigningrequests  csr   certificates.k8s.io/v1   false  CertificateSigningRequest

# Search for anything with "policy"
k api-resources | grep policy

# Get just the names
k api-resources -o name
k api-resources -o name | grep -i "net"  # Find network-related
```

**Using discovered resources**:
```bash
# Once you find it exists, create/get/describe it
k get certificatesigningrequest
k describe certificatesigningrequest <name>
k get csr  # short name!

# Get YAML definition
k explain certificatesigningrequest
k explain certificatesigningrequest.spec

# Get by API version
k get certificates.k8s.io/v1 my-cert
```

---

## 🏗️ Static Pod Manifests

**Control plane runs as static pods. Know this cold.**

**Where they live**:
```bash
/etc/kubernetes/manifests/
  ├── kube-apiserver.yaml
  ├── kube-controller-manager.yaml
  ├── kube-scheduler.yaml
  └── etcd.yaml
```

**Kubelet watches this directory** - any changes auto-restart pod in ~30s.

### Reading Static Pod Manifests
```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Look for:
# 1. Image version
image: registry.k8s.io/kube-apiserver:v1.35.0

# 2. Volume mounts (where certs, configs come from)
volumeMounts:
- name: k8s-certs
  mountPath: /etc/kubernetes/pki
  readOnly: true
- name: ca-certs
  mountPath: /etc/ssl/certs
  readOnly: true

# 3. Volumes (maps host dirs into container)
volumes:
- name: k8s-certs
  hostPath:
    path: /etc/kubernetes/pki
    type: DirectoryOrCreate

# 4. Command arguments (behavior)
command:
- kube-apiserver
- --advertise-address=10.0.0.5
- --client-ca-file=/etc/kubernetes/pki/ca.crt
- --etcd-servers=https://127.0.0.1:2379
- --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
```

### Common Static Pod Issues

**API server won't start**:
```bash
# Check manifest for typos
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Common errors:
# - Wrong flag name: --kubeconfig vs --kubelet-config
# - Wrong path: /etc/kubernetes/pki/ca.crt doesn't exist
# - Wrong image: typo in version
# - Wrong etcd endpoint: pointing to wrong IP

# Check logs
journalctl -u kubelet | grep apiserver

# Fix and save (kubelet restarts pod in ~30s)
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Scheduler/Controller Manager not running**:
```bash
# These are also static pods
cat /etc/kubernetes/manifests/kube-scheduler.yaml
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Common: wrong kubeconfig path
# Should be: /etc/kubernetes/scheduler.conf, /etc/kubernetes/controller-manager.conf

# Check if files exist
ls -la /etc/kubernetes/*.conf

# Verify kubeconfig points to correct server
grep "server:" /etc/kubernetes/scheduler.conf
```

---

## 🔄 kubeconfig & Context Management

**kubeconfig = how kubectl knows where API server is + auth creds**

### kubeconfig Structure
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi... (base64 CA cert)
    server: https://10.0.0.5:6443  # API server address
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin  # Links to user below
  name: kubernetes-admin@kubernetes  # Context name
current-context: kubernetes-admin@kubernetes  # Active context
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTi... (admin cert)
    client-key-data: LS0tLS1QUklWQVRFIEtFWS... (admin key)
```

### Manage kubeconfig
```bash
# View current context
k config current-context

# View all contexts
k config get-contexts

# Switch context
k config use-context kubernetes-admin@kubernetes

# Set context for a command
k --context=<context> get pods

# View kubeconfig
cat ~/.kube/config

# Multiple kubeconfigs
export KUBECONFIG=/path/to/config1:/path/to/config2
k config view  # Shows merged view

# Create kubeconfig entry for new cluster
k config set-cluster my-cluster --server=https://10.0.0.5:6443 \
  --certificate-authority=/path/to/ca.crt

k config set-credentials my-user \
  --client-certificate=/path/to/client.crt \
  --client-key=/path/to/client.key

k config set-context my-context --cluster=my-cluster --user=my-user
k config use-context my-context

# View specific cluster details
k config get-clusters
k config view --flatten | grep -A 5 "cluster: kubernetes"
```

### Troubleshoot kubeconfig Issues
```bash
# kubectl won't connect
k cluster-info
# Error: error validating the server certificate

# Check if server is reachable
curl -k https://API-SERVER-IP:6443

# Check kubeconfig syntax
k config view
# Should show clusters, contexts, users

# Check cert validity
openssl x509 -in /etc/kubernetes/admin.conf -noout -text 2>/dev/null
# If embedded cert, decode first:
grep client-certificate-data ~/.kube/config | cut -d' ' -f2 | base64 -d | \
  openssl x509 -noout -text

# Check if context exists
k config get-contexts | grep <name>

# Verify cert CA matches cluster CA
grep certificate-authority ~/.kube/config
ls /etc/kubernetes/pki/ca.crt
# Should match!
```

---

## 🚀 kubeadm - Cluster Installation & Upgrades

### Init Control Plane
```bash
# 1. Prepare (on control plane node)
# - Install kubeadm, kubectl, kubelet
# - Disable swap

# 2. Initialize
sudo kubeadm init \
  --apiserver-advertise-address=10.0.0.5 \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.35.0

# Output includes kubeadm join command (SAVE IT!)

# 3. Setup kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 4. Install CNI
k apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 5. Verify
k get nodes
k get pods -n kube-system  # Should see kube-flannel, coredns, etc.
```

### Join Worker Node
```bash
# On worker, run the kubeadm join command from init output:
sudo kubeadm join 10.0.0.5:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Verify from control plane
k get nodes
k describe node <worker>
```

### Upgrade Cluster
```bash
# ALWAYS on control plane FIRST, then workers

# 1. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.35.0-*
sudo apt-mark hold kubeadm

# 2. Plan upgrade
sudo kubeadm upgrade plan

# 3. Apply
sudo kubeadm upgrade apply v1.35.0

# 4. Upgrade kubelet/kubectl (control plane)
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.35.0-* kubectl=1.35.0-*
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 5. On workers (from control plane)
k drain <worker> --ignore-daemonsets --delete-emptydir-data

# 6. On each worker node (SSH)
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubeadm=1.35.0-* kubelet=1.35.0-* kubectl=1.35.0-*
sudo apt-mark hold kubeadm kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 7. Uncordon worker
k uncordon <worker>
```

---

## 💾 etcd - Backup & Restore

**etcd = cluster database. Back it up!**

### etcdctl Commands
```bash
# Snapshot backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-backup.db

# Restore from snapshot
# 1. Stop API server and control plane
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored \
  --initial-cluster=cp-0=https://10.0.0.5:2380 \
  --initial-advertise-peer-urls=https://10.0.0.5:2380 \
  --name=cp-0

# 3. Move restored data
sudo mv /var/lib/etcd /var/lib/etcd-old
sudo mv /var/lib/etcd-restored /var/lib/etcd

# 4. Restart API server
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Verify
watch k get pods -n kube-system
```

**Find etcdctl on exam**:
```bash
# etcdctl might not be in PATH
which etcdctl
# Usually: /usr/local/bin/etcdctl

# Or in container
k exec -it etcd-<nodename> -n kube-system -- sh
# Inside: etcdctl snapshot save /backup.db ...
```

---

## 🐛 Node Troubleshooting

### Node NotReady
```bash
k describe node <node>
# Check Conditions: what's False/Unknown?

# Solutions by condition:
# Ready=False: kubelet not running, certs expired, network issue
# DiskPressure=True: clean disk (crictl rmi, docker prune)
# MemoryPressure=True: find OOMing pods (k top pods)
# PIDPressure=True: too many processes (crictl ps | wc -l)

# Debug on node
systemctl status kubelet
free -h
df -h
crictl ps | wc -l
```

### Drain/Cordon
```bash
# Cordon = mark NotReady, no new pods scheduled
k cordon <node>

# Drain = evict existing pods gracefully
k drain <node> --ignore-daemonsets --delete-emptydir-data

# Uncordon = make Ready again
k uncordon <node>

# Common exam scenario:
# "Fix node, then make ready for workload"
# Steps:
# 1. k get nodes (find NotReady)
# 2. SSH to node
# 3. systemctl start kubelet
# 4. From control plane: k uncordon <node>
```

---

## 🌐 Control Plane Component Troubleshooting

### API Server Down
```bash
# Symptoms: kubectl commands fail with "connection refused"

# 1. Check if pod exists
k get pods -n kube-system -l component=kube-apiserver
# OR
k get pod -n kube-system kube-apiserver-<nodename>

# 2. Check logs
k logs -n kube-system kube-apiserver-<nodename>

# 3. Check manifest
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Common issues:
# - Wrong flag: --kubeconfig instead of --kubelet-config
# - Missing volume mount for certs
# - Wrong server address in etcd flag
# - Cert expired (check /etc/kubernetes/pki/apiserver.crt dates)

# 4. Fix and wait (kubelet restarts in ~30s)
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
watch k get pods -n kube-system | grep apiserver
```

### Scheduler Down
```bash
# Symptoms: pods stuck in Pending

# 1. Check if running
k get pods -n kube-system -l component=kube-scheduler

# 2. Check logs
k logs -n kube-system kube-scheduler-<nodename>

# 3. Check manifest
sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Look for: --kubeconfig=/etc/kubernetes/scheduler.conf
# Verify: ls /etc/kubernetes/scheduler.conf

# 4. If can't connect to API server
# Check kubeconfig:
grep server /etc/kubernetes/scheduler.conf
# Should point to: https://127.0.0.1:6443 OR https://<control-plane-ip>:6443
```

### Controller Manager Down
```bash
# Symptoms: deployments/replicasets not being managed

# 1. Check pod
k get pods -n kube-system -l component=kube-controller-manager

# 2. Logs
k logs -n kube-system kube-controller-manager-<nodename>

# 3. Manifest
sudo cat /etc/kubernetes/manifests/kube-controller-manager.yaml
# Look for: --kubeconfig=/etc/kubernetes/controller-manager.conf
# Verify: ls /etc/kubernetes/controller-manager.conf
```

### etcd Down (cluster can't function)
```bash
# Symptoms: API server can't start, all kubectl commands fail

# 1. Check pod/container
k get pods -n kube-system etcd-<nodename>
# OR
crictl ps | grep etcd

# 2. Logs
k logs -n kube-system etcd-<nodename>
# OR
journalctl -u kubelet | grep etcd

# 3. Check data dir
ls -la /var/lib/etcd/

# Common issues:
# - Disk full: df -h
# - Corrupted data: restore from backup
# - Wrong certs: openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -dates

# 4. Check manifest
cat /etc/kubernetes/manifests/etcd.yaml
# Verify volumes point to /var/lib/etcd
```

---

## 🔗 Service & Networking Troubleshooting

### Service has no endpoints
```bash
k get svc my-service
k get endpoints my-service
# Endpoints should list pod IPs

# If empty:
# 1. Check service selector
k describe svc my-service | grep Selector

# 2. Check pod labels
k get pods --show-labels
# Labels must match service selector!

# 3. Fix: relabel pods
k label pod <pod> app=myapp

# 4. Verify
k get endpoints my-service  # Should populate
```

### Pod can't reach service
```bash
# From pod, test connectivity
k exec <pod> -- curl http://my-service:80

# If timeout:
# 1. Check service exists and has endpoints
k get svc my-service
k get endpoints my-service

# 2. Check DNS
k exec <pod> -- nslookup my-service.default.svc.cluster.local

# 3. Check CoreDNS is running
k get pods -n kube-system -l k8s-app=kube-dns

# 4. Check NetworkPolicy
k get networkpolicies
k describe networkpolicy <policy>
# Check ingress/egress rules

# 5. Check kube-proxy
k get pods -n kube-system -l k8s-app=kube-proxy
k logs -n kube-system -l k8s-app=kube-proxy
```

---

## 📊 Resource Limits & Quotas

### Request vs Limit
```yaml
resources:
  requests:
    memory: "256Mi"   # Minimum needed, scheduler uses this
    cpu: "100m"       # If not set, defaults to 0 (best effort)
  limits:
    memory: "512Mi"   # Max allowed, pod killed if exceeded (OOMKilled)
    cpu: "500m"       # Max allowed, throttled if exceeded
```

### ResourceQuota (namespace-level)
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota
  namespace: myapp
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    pods: "10"
    services: "5"
```

```bash
# Check quota
k describe quota -n myapp

# If pod creation fails with quota error
k describe pod <pod> | grep Message

# Check namespace usage
k describe quota -n myapp
```

---

## 🔐 RBAC Deep Dive

### Role vs ClusterRole
```bash
# Role = namespace-scoped
k create role my-role --verb=get,list --resource=pods,services

# ClusterRole = cluster-scoped
k create clusterrole my-crole --verb=get,list --resource=nodes

# Check what's available
k api-resources
# NAMESPACED column: true=Role/RoleBinding, false=ClusterRole/ClusterRoleBinding
```

### RoleBinding vs ClusterRoleBinding
```bash
# Bind role to user in namespace
k create rolebinding my-binding --role=my-role --user=alice

# Bind clusterrole to user cluster-wide
k create clusterrolebinding my-cbinding --clusterrole=my-crole --user=alice

# Verify
k get rolebindings
k get clusterrolebindings
k describe rolebinding my-binding
```

### Test RBAC
```bash
# Create user (self-signed cert)
openssl req -new -newkey rsa:2048 -nodes \
  -out alice.csr -keyout alice.key -subj "/CN=alice/O=users"

# Sign with cluster CA
openssl x509 -req -days 365 -in alice.csr \
  -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out alice.crt

# Add to kubeconfig
k config set-credentials alice --client-certificate=alice.crt --client-key=alice.key
k config set-context alice --cluster=kubernetes --user=alice
k config use-context alice

# Test (should fail without RBAC)
k get pods
# Error: pods is forbidden

# Grant access
k create role pod-reader --verb=get,list --resource=pods
k create rolebinding alice-pod-reader --role=pod-reader --user=alice

# Test again
k config use-context alice
k get pods  # Should work now!
```

---

## 📝 Common Exam Patterns

### Broken Cluster - Diagnosis Flow
```bash
# 1. What's broken?
k get nodes
k get pods -n kube-system

# 2. Isolate
# - Is it control plane? (can't kubectl)
# - Is it specific node? (NotReady)
# - Is it specific pod? (CrashLoopBackOff)

# 3. If control plane broken
# SSH to control plane
systemctl status kubelet
journalctl -u kubelet -f
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# 4. If node broken
# SSH to node
systemctl status kubelet
journalctl -u kubelet -f
k describe node <node>

# 5. If pod broken
k describe pod <pod>
k logs <pod>
k logs <pod> --previous
```

### Fix Certificate Path Issues
```bash
# Symptom: apiserver won't start, x509 errors in logs

# 1. Check manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep "ca-file\|cert\|key"

# 2. Verify files exist
ls /etc/kubernetes/pki/ca.crt
ls /etc/kubernetes/pki/apiserver.crt
ls /etc/kubernetes/pki/apiserver.key

# 3. Check permissions
ls -la /etc/kubernetes/pki/
# kubelet runs as root, should be readable

# 4. Fix path in manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# --client-ca-file=/etc/kubernetes/pki/ca.crt (correct)
# NOT /etc/pki/ca.crt (wrong)

# 5. Restart
sudo systemctl restart kubelet
```

### Fix Wrong kubeconfig Paths
```bash
# Symptoms: kubelet can't connect, connection refused

# 1. Check kubeconfig points to right place
cat /etc/kubernetes/kubelet.conf | grep "server:"
# Should be: https://10.0.0.5:6443 (control plane IP)
# NOT: https://127.0.0.1:6443 (localhost - only works on control plane)

# 2. Fix if needed
sudo sed -i 's|server: https://127.0.0.1:6443|server: https://10.0.0.5:6443|' /etc/kubernetes/kubelet.conf

# 3. Restart kubelet
sudo systemctl restart kubelet

# 4. Verify
systemctl status kubelet
k get nodes
```

