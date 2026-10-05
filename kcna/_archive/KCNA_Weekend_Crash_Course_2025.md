# KCNA Weekend Crash Course 2025
## Your Path to 100% - Complete Study Guide

**Target Audience**: Mid-CKA level engineers speedrunning KCNA  
**Time Required**: 2-3 days of focused study  
**Goal**: Comprehensive conceptual mastery + 100% exam readiness

---

## 📋 Exam Overview

**Format**: Multiple Choice (60 questions)  
**Duration**: 90 minutes  
**Passing Score**: 75% (aim for 90%+)  
**Cost**: $250 (includes 1 free retake)  
**Validity**: 3 years  
**Environment**: Online, remotely proctored

**Key Differences from CKA**:
- NO hands-on terminal work
- NO YAML writing
- Pure conceptual understanding
- Broader scope (CNCF ecosystem, not just K8s)
- Easier but requires breadth of knowledge

---

## 🎯 Exam Domains & Weights

| Domain | Weight | Focus |
|--------|--------|-------|
| **Kubernetes Fundamentals** | 46% | Architecture, API objects, kubectl basics |
| **Container Orchestration** | 22% | Deployments, scheduling, scaling, security |
| **Cloud Native Architecture** | 16% | Microservices, 12-factor, service mesh, serverless |
| **Cloud Native Observability** | 8% | Monitoring, logging, tracing (Prometheus, etc.) |
| **Cloud Native Application Delivery** | 8% | CI/CD, GitOps, Helm, Argo |

---

## 📚 Domain 1: Kubernetes Fundamentals (46%)

### 1.1 Kubernetes Architecture

**Control Plane Components**:
- **kube-apiserver**: Front-end for K8s control plane, exposes API
- **etcd**: Distributed key-value store for all cluster data
- **kube-scheduler**: Assigns Pods to nodes based on resource requirements
- **kube-controller-manager**: Runs controllers (Deployment, ReplicaSet, etc.)
- **cloud-controller-manager**: Integrates with cloud provider APIs

**Node Components**:
- **kubelet**: Agent that ensures containers run in Pods
- **kube-proxy**: Network proxy implementing Service networking
- **Container runtime**: Runs containers (containerd, CRI-O, Docker Engine)

**Edge Cases**:
- etcd is the ONLY stateful component in control plane
- Scheduler doesn't actually start Pods - kubelet does
- kube-proxy doesn't actually proxy traffic in default iptables mode

### 1.2 Kubernetes API Primitives

**Core Workload Resources**:

**Pod**:
- Smallest deployable unit
- Can contain 1+ containers (usually 1)
- Shares network namespace (same IP, localhost communication)
- Ephemeral - no pod should be irreplaceable

**ReplicaSet**:
- Maintains desired number of Pod replicas
- Selector-based (matches labels)
- Usually managed by Deployment (don't create directly)

**Deployment**:
- Declarative updates for Pods and ReplicaSets
- Rolling updates, rollbacks
- Most common way to run stateless apps

**StatefulSet**:
- For stateful applications
- Stable, unique network identifiers (pod-0, pod-1)
- Stable persistent storage
- Ordered, graceful deployment and scaling

**DaemonSet**:
- Ensures copy of Pod runs on all (or selected) nodes
- Use cases: log collectors, monitoring agents, storage daemons

**Job**:
- Creates Pods that run to completion
- For batch processing, one-off tasks

**CronJob**:
- Creates Jobs on a schedule (cron format)
- For periodic tasks

**Service Types**:

1. **ClusterIP** (default):
   - Internal cluster IP
   - Only accessible within cluster
   - Most common for inter-service communication

2. **NodePort**:
   - Exposes on static port on each node
   - External access via `<NodeIP>:<NodePort>`
   - Port range: 30000-32767

3. **LoadBalancer**:
   - Creates external load balancer (cloud provider)
   - Gets external IP
   - Automatically creates NodePort and ClusterIP

4. **ExternalName**:
   - Maps Service to DNS name
   - No proxying, just DNS CNAME

**Storage**:

**Volume Types (common)**:
- **emptyDir**: Ephemeral, Pod lifetime
- **hostPath**: Mount from node filesystem (avoid in production)
- **configMap**: Inject config data
- **secret**: Inject sensitive data
- **persistentVolumeClaim**: Request PV storage

**PersistentVolume (PV)**: Cluster resource provisioned by admin  
**PersistentVolumeClaim (PVC)**: Request for storage by user  
**StorageClass**: Dynamic provisioning blueprint

**Access Modes**:
- RWO (ReadWriteOnce): Single node mount
- ROX (ReadOnlyMany): Multiple nodes read-only
- RWX (ReadWriteMany): Multiple nodes read-write

**Configuration**:

**ConfigMap**:
- Store non-sensitive config data
- Key-value pairs
- Can be env vars or mounted as files

**Secret**:
- Store sensitive data (passwords, tokens, keys)
- Base64 encoded (NOT encrypted by default)
- Types: Opaque, TLS, Docker registry, etc.

**Namespace**:
- Virtual clusters within physical cluster
- Resource isolation and organization
- Default namespaces: default, kube-system, kube-public, kube-node-lease

### 1.3 kubectl Basics (Conceptual)

You won't use kubectl in exam, but know what these do:

```bash
# Create resources
kubectl create deployment
kubectl expose
kubectl run

# Read resources
kubectl get [resource]
kubectl describe [resource]
kubectl logs [pod]

# Update resources
kubectl edit
kubectl scale
kubectl set image
kubectl rollout (status|history|undo)

# Delete resources
kubectl delete [resource]

# Context/namespace
kubectl config use-context
kubectl config set-context --current --namespace
```

**Edge Cases**:
- `kubectl apply` vs `kubectl create`: apply is declarative, can update
- `kubectl logs` doesn't work on Deployments directly (need Pod name)
- `kubectl exec` runs commands in containers, not Pods

---

## 🔧 Domain 2: Container Orchestration (22%)

### 2.1 Container Runtimes

**What is a Container Runtime?**
- Software that executes containers and manages container images
- Implements CRI (Container Runtime Interface)

**Common Runtimes**:
- **containerd**: Industry standard, from Docker, now standalone (most popular)
- **CRI-O**: Lightweight, OCI-compliant, built for K8s
- **Docker Engine**: Full Docker daemon (deprecated in K8s 1.24+)

**Container Runtime Interface (CRI)**:
- Standard for container runtimes to integrate with kubelet
- Abstracts runtime implementation

**Edge Cases**:
- Docker is NOT a container runtime for K8s anymore (use containerd)
- containerd is what Docker uses under the hood
- CRI-O is K8s-specific, containerd is general-purpose

### 2.2 Container Networking

**CNI (Container Network Interface)**:
- Standard for network plugins
- Handles Pod networking, IP allocation

**Popular CNI Plugins**:
- **Calico**: L3 networking, network policy enforcement
- **Flannel**: Simple overlay network
- **Weave Net**: Mesh network
- **Cilium**: eBPF-based, high performance, security

**Network Concepts**:
- Each Pod gets unique IP (shared by containers)
- Pod-to-Pod communication without NAT
- Services provide stable endpoints

**Service Discovery**:
- DNS (CoreDNS)
- Environment variables
- Service names resolve to ClusterIP

### 2.3 Scheduling

**How Scheduling Works**:
1. User creates Pod
2. API server stores in etcd
3. Scheduler watches for unscheduled Pods
4. Scheduler selects node based on filters and priorities
5. Binds Pod to node
6. kubelet on node starts container

**Scheduling Concepts**:

**Node Selector**:
- Simple key-value label matching
- `nodeSelector: disktype=ssd`

**Node Affinity**:
- More expressive than nodeSelector
- Required vs preferred rules
- In/NotIn operators

**Taints and Tolerations**:
- Taints: Mark nodes to repel Pods
- Tolerations: Allow Pods to schedule on tainted nodes
- Effects: NoSchedule, PreferNoSchedule, NoExecute

**Pod Affinity/Anti-Affinity**:
- Schedule Pods together (affinity) or apart (anti-affinity)
- Use cases: co-locate related services, spread for HA

**Resource Requests and Limits**:
- Requests: Minimum guaranteed resources (used for scheduling)
- Limits: Maximum resources Pod can use
- CPU: measured in cores (1000m = 1 core)
- Memory: bytes (Ki, Mi, Gi)

### 2.4 Security

**Authentication**:
- X.509 client certificates
- Bearer tokens
- Service Account tokens
- External auth providers (OIDC, webhooks)

**Authorization**:
- RBAC (Role-Based Access Control) - most common
- ABAC (Attribute-Based)
- Webhook
- Node authorization

**RBAC Components**:
- **Role**: Permissions within namespace
- **ClusterRole**: Cluster-wide permissions
- **RoleBinding**: Grants Role to subjects in namespace
- **ClusterRoleBinding**: Grants ClusterRole cluster-wide

**Subjects**: User, Group, ServiceAccount

**Pod Security**:

**Pod Security Standards** (replaces PodSecurityPolicy):
- **Privileged**: Unrestricted
- **Baseline**: Minimally restrictive
- **Restricted**: Heavily restricted, best practices

**Security Context**:
- runAsUser, runAsGroup
- fsGroup
- capabilities (add/drop)
- readOnlyRootFilesystem
- allowPrivilegeEscalation

**Network Policies**:
- Firewall rules for Pods
- Ingress (incoming) and Egress (outgoing)
- Label-based selectors
- Default deny all, then allow specific

**Edge Cases**:
- ServiceAccounts are for Pods, not humans
- RBAC is additive (no deny rules)
- Network policies are namespace-scoped
- Need CNI plugin that supports NetworkPolicy (Calico, Cilium)

---

## 🏗️ Domain 3: Cloud Native Architecture (16%)

### 3.1 Cloud Native Principles

**Cloud Native Definition (CNCF)**:
Technologies that empower organizations to build and run scalable applications in modern, dynamic environments (public, private, hybrid clouds). Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.

**Key Characteristics**:
- **Microservices**: Small, independent services
- **Containerization**: Package with dependencies
- **Dynamic orchestration**: K8s, automated scaling
- **DevOps culture**: Collaboration, automation
- **Continuous delivery**: Automated deployment pipelines

### 3.2 Microservices Architecture

**Benefits**:
- Independent deployment
- Technology diversity
- Fault isolation
- Easy scaling

**Challenges**:
- Distributed complexity
- Network latency
- Data consistency
- Testing difficulty

**Communication Patterns**:
- Synchronous: REST, gRPC
- Asynchronous: Message queues, event streaming

### 3.3 12-Factor App Methodology

Must know these principles:

1. **Codebase**: One codebase tracked in version control
2. **Dependencies**: Explicitly declare and isolate dependencies
3. **Config**: Store config in environment
4. **Backing Services**: Treat backing services as attached resources
5. **Build, Release, Run**: Strictly separate build and run stages
6. **Processes**: Execute app as stateless processes
7. **Port Binding**: Export services via port binding
8. **Concurrency**: Scale out via process model
9. **Disposability**: Fast startup and graceful shutdown
10. **Dev/Prod Parity**: Keep development and production similar
11. **Logs**: Treat logs as event streams
12. **Admin Processes**: Run admin tasks as one-off processes

### 3.4 Service Mesh

**What is a Service Mesh?**
- Infrastructure layer for service-to-service communication
- Handles routing, load balancing, security, observability
- Sidecar proxy pattern (proxy container alongside app container)

**Popular Service Meshes**:
- **Istio**: Feature-rich, complex, uses Envoy proxy
- **Linkerd**: Lightweight, simple, Rust-based
- **Consul**: HashiCorp, service discovery + mesh

**Features**:
- Traffic management (routing, load balancing)
- Security (mTLS, authentication)
- Observability (metrics, tracing)
- Resilience (retries, circuit breaking, timeouts)

### 3.5 Serverless

**Serverless Computing**:
- Run code without managing servers
- Event-driven, auto-scaling
- Pay per execution

**K8s Serverless**:
- **Knative**: K8s-based serverless platform
  - Serving: Request-driven auto-scaling
  - Eventing: Event-driven architecture

**Functions as a Service (FaaS)**:
- OpenFaaS
- Kubeless
- Fission

### 3.6 API Gateway

**Purpose**:
- Single entry point for clients
- Request routing, composition
- Authentication, rate limiting
- Protocol translation

**Popular in K8s**:
- **Kong**: Plugin-based, Lua extensible
- **Ambassador**: Envoy-based, declarative
- **NGINX Ingress**: Simple, widely used

### 3.7 Autoscaling

**Types**:

**Horizontal Pod Autoscaler (HPA)**:
- Scales number of Pods
- Based on CPU, memory, custom metrics
- Requires metrics-server

**Vertical Pod Autoscaler (VPA)**:
- Adjusts Pod resource requests/limits
- Requires Pod restart (most modes)

**Cluster Autoscaler**:
- Scales number of nodes
- Cloud provider integration

**KEDA (Kubernetes Event-Driven Autoscaling)**:
- Scale based on events (queue length, etc.)
- Extends HPA

**Edge Cases**:
- HPA and VPA shouldn't be used together on same metric
- Cluster autoscaler works with node groups/pools
- HPA cooldown/delay to prevent flapping

---

## 📊 Domain 4: Cloud Native Observability (8%)

### 4.1 Monitoring

**Prometheus** (CNCF graduated):
- Time-series database
- Pull-based metric collection
- PromQL query language
- Service discovery
- Alerting (with Alertmanager)

**Key Concepts**:
- **Metrics**: Numerical measurements over time
- **Labels**: Key-value pairs for dimensionality
- **Exporters**: Expose metrics from services
- **Targets**: Endpoints Prometheus scrapes

**Metric Types**:
- Counter: Cumulative, only increases
- Gauge: Can go up or down
- Histogram: Observations in buckets
- Summary: Like histogram, client-side quantiles

**Grafana**:
- Visualization and dashboards
- Connects to Prometheus (and others)
- Alerting, annotations

### 4.2 Logging

**Logging Strategies**:
- **Node-level**: DaemonSet log collector on each node
- **Sidecar**: Logging container per Pod
- **Application-level**: App logs directly to backend

**EFK/ELK Stack**:
- **Elasticsearch**: Store and index logs
- **Fluentd/Logstash**: Collect and transform logs
- **Kibana**: Visualize and search logs

**Fluentd** (CNCF graduated):
- Unified logging layer
- Plugin-based architecture
- Buffers, transforms, routes logs

**Loki** (Grafana):
- Like Prometheus but for logs
- Indexes labels, not full text
- Lightweight, cost-effective

### 4.3 Distributed Tracing

**Why Tracing?**
- Understand request flow across microservices
- Identify bottlenecks and failures
- Measure latency

**OpenTelemetry** (CNCF):
- Vendor-neutral APIs and SDKs
- Unified standard for traces, metrics, logs
- Replaced OpenTracing and OpenCensus

**Jaeger** (CNCF graduated):
- Distributed tracing platform
- Trace collection, storage, visualization
- Sampling strategies

**Key Concepts**:
- **Trace**: End-to-end journey of request
- **Span**: Single operation within trace
- **Context Propagation**: Passing trace ID between services

### 4.4 Observability in K8s

**Metrics-server**:
- Collects resource metrics (CPU, memory)
- Required for HPA
- kubectl top nodes/pods

**kube-state-metrics**:
- Generates metrics about K8s objects
- Different from metrics-server (cluster state vs resource usage)

**Edge Cases**:
- Prometheus pulls, not pushes (except Pushgateway for batch jobs)
- Logging to stdout/stderr is K8s best practice
- Distributed tracing requires instrumentation in apps

---

## 🚀 Domain 5: Cloud Native Application Delivery (8%)

### 5.1 CI/CD Concepts

**Continuous Integration (CI)**:
- Frequent code integration
- Automated builds and tests
- Fast feedback

**Continuous Delivery (CD)**:
- Automated deployment to staging
- Manual approval for production

**Continuous Deployment**:
- Fully automated to production

**CI/CD Best Practices**:
- Version control everything
- Automate tests
- Keep builds fast
- Monitor deployments
- Rollback capability

### 5.2 CI/CD Tools (CNCF & Popular)

**Jenkins**:
- Traditional, widely used
- Plugin ecosystem
- Pipelines as code (Jenkinsfile)

**GitLab CI/CD**:
- Built into GitLab
- .gitlab-ci.yml configuration
- Auto DevOps

**GitHub Actions**:
- Integrated with GitHub
- Workflow YAML files
- Marketplace for actions

**Tekton** (CNCF):
- K8s-native CI/CD
- Building blocks (Tasks, Pipelines)
- Cloud-agnostic

**Flux** (CNCF):
- GitOps operator for K8s
- Syncs cluster state with Git
- Automated image updates

**Argo CD** (CNCF):
- Declarative GitOps for K8s
- UI for application management
- Sync strategies, rollbacks

### 5.3 GitOps

**GitOps Principles**:
1. Declarative: System described declaratively
2. Versioned: Desired state stored in Git
3. Pulled automatically: Agents pull changes
4. Continuously reconciled: Ensure actual = desired

**Benefits**:
- Single source of truth (Git)
- Audit trail
- Easy rollbacks
- Disaster recovery

**GitOps Tools**:
- Flux
- Argo CD
- Jenkins X

### 5.4 Package Management

**Helm** (CNCF graduated):
- K8s package manager
- Charts: collection of K8s manifests
- Templating with values
- Release management

**Key Concepts**:
- **Chart**: Package of K8s resources
- **Release**: Instance of chart running in cluster
- **Repository**: Store for charts
- **Values**: Configuration parameters

**Helm Commands (conceptual)**:
- helm install
- helm upgrade
- helm rollback
- helm list
- helm uninstall

**Kustomize**:
- K8s-native configuration management
- Overlay-based customization
- Built into kubectl (kubectl apply -k)

**Edge Cases**:
- Helm 2 vs 3: Tiller removed in v3
- Kustomize doesn't use templates (overlays instead)
- GitOps can use Helm charts

---

## 🌐 CNCF Landscape & Projects

**CRITICAL**: Know which CNCF project solves which problem!

### Graduated Projects (Most Mature)

**Orchestration**:
- **Kubernetes**: Container orchestration

**Container Runtime**:
- **containerd**: Container runtime

**Networking**:
- **Envoy**: Service proxy, used by Istio, Ambassador
- **CoreDNS**: DNS server for K8s

**Monitoring**:
- **Prometheus**: Metrics and alerting

**Logging**:
- **Fluentd**: Unified logging

**Tracing**:
- **Jaeger**: Distributed tracing

**Storage**:
- **Rook**: Cloud-native storage orchestrator

**Service Mesh**:
- **Linkerd**: Lightweight service mesh

**Package Management**:
- **Helm**: K8s package manager

**Security**:
- **Falco**: Runtime security, threat detection

### Incubating Projects (Growing)

**CI/CD**:
- **Argo**: GitOps, workflows, events, rollouts
- **Flux**: GitOps operator

**Observability**:
- **OpenTelemetry**: Unified observability framework
- **Thanos**: Prometheus long-term storage

**Networking**:
- **Cilium**: eBPF-based networking and security
- **Contour**: Ingress controller (Envoy-based)

**Security**:
- **OPA (Open Policy Agent)**: Policy-based control
- **SPIFFE/SPIRE**: Workload identity

**Serverless**:
- **Knative**: K8s-based serverless

**Registry**:
- **Harbor**: Container registry

**Service Mesh**:
- **Istio**: Feature-rich service mesh (recently graduated)

### Sandbox Projects (Early Stage)

**Many projects here, know categories**:
- Service mesh: Kuma, Service Mesh Interface
- CI/CD: Keptn, Tekton
- Security: Notary, in-toto
- Observability: Pixie

### Project Selection Guide

**Which project for which problem?**

| Problem | CNCF Project |
|---------|--------------|
| Container orchestration | Kubernetes |
| Container runtime | containerd, CRI-O |
| Service proxy | Envoy |
| Service mesh | Istio, Linkerd |
| DNS | CoreDNS |
| Ingress controller | Contour, NGINX Ingress |
| Networking | Calico, Cilium, Flannel, Weave |
| Metrics/monitoring | Prometheus |
| Logging | Fluentd |
| Distributed tracing | Jaeger, OpenTelemetry |
| Storage orchestration | Rook |
| Package management | Helm |
| GitOps | Argo CD, Flux |
| CI/CD pipelines | Tekton, Argo Workflows |
| Policy enforcement | OPA |
| Runtime security | Falco |
| Container registry | Harbor |
| Serverless | Knative |
| Workload identity | SPIFFE/SPIRE |

**Edge Cases**:
- Envoy is a proxy, not a full service mesh (Istio uses Envoy)
- OpenTelemetry merges OpenTracing and OpenCensus
- Prometheus doesn't do long-term storage (use Thanos or Cortex)
- OPA is policy engine, doesn't enforce (needs integration)

---

## 🎓 Study Strategy for Weekend Speedrun

### Day 1: Morning (4 hours)
1. **Kubernetes Fundamentals** (2.5 hours)
   - Architecture deep dive
   - All workload resources
   - Services and networking
   
2. **Container Orchestration** (1.5 hours)
   - Runtimes, CNI
   - Scheduling concepts
   - Security (RBAC, Pod security)

### Day 1: Afternoon (3 hours)
3. **Cloud Native Architecture** (2 hours)
   - 12-factor app
   - Microservices patterns
   - Service mesh, serverless
   - API gateway, autoscaling

4. **CNCF Landscape Review** (1 hour)
   - Map projects to categories
   - Understand graduated vs incubating
   - Problem → Solution mapping

### Day 2: Morning (3 hours)
5. **Observability** (1.5 hours)
   - Prometheus ecosystem
   - Logging strategies (Fluentd, Loki)
   - Tracing (Jaeger, OpenTelemetry)

6. **Application Delivery** (1.5 hours)
   - CI/CD concepts and tools
   - GitOps principles
   - Helm and Kustomize

### Day 2: Afternoon (4 hours)
7. **Practice Exams** (4 hours)
   - Take 3-4 full practice exams
   - Review wrong answers thoroughly
   - Identify weak areas
   - Retake weak domain questions

### Day 3: Review & Edge Cases (6 hours)
8. **Weak Area Deep Dive** (2 hours)
9. **CNCF Project Quiz** (1 hour)
10. **Edge Cases & Tricky Concepts** (2 hours)
11. **Final Practice Exam** (1 hour)

---

## 🔥 Edge Cases & Gotchas

### Kubernetes Specific

1. **Pods vs Containers**:
   - Pods can have multiple containers (but usually don't)
   - Init containers run before app containers
   - Sidecar containers run alongside app containers

2. **Services Don't Select Pods Directly**:
   - Services select by labels
   - Pod labels can change, Service keeps working

3. **StatefulSet Pod Names**:
   - Predictable: web-0, web-1, web-2
   - Ordered creation/deletion

4. **DaemonSet Doesn't Respect Replicas**:
   - One Pod per matching node automatically

5. **Job Completion**:
   - Pod exit code 0 = success
   - Can have multiple completions, parallelism

6. **NodePort Range**:
   - 30000-32767 by default
   - Can be changed in API server config

7. **PVC Binding**:
   - Waits for matching PV or StorageClass
   - Can be Pending indefinitely

8. **ConfigMap vs Secret**:
   - Secrets are base64 (not encrypted in etcd by default)
   - Both can be env vars or volumes

9. **Network Policy Default**:
   - If no policies exist, all traffic allowed
   - Once policy exists, default deny

10. **RBAC is Additive**:
    - No deny rules
    - Union of all permissions

### Cloud Native Architecture

11. **Service Mesh Overhead**:
    - Adds latency (proxy hop)
    - Resource usage (sidecar containers)

12. **Serverless Cold Starts**:
    - First request slow
    - Knative has scale-to-zero

13. **12-Factor ≠ Microservices**:
    - 12-factor can apply to monoliths
    - Microservices benefit from 12-factor

14. **Microservices Data**:
    - Each service owns its data
    - No shared databases (best practice)

### Observability

15. **Prometheus Pull Model**:
    - Scrapes targets, doesn't receive pushes
    - Pushgateway exception for batch jobs

16. **Logging to stdout/stderr**:
    - K8s best practice
    - Let log collector handle routing

17. **Metrics vs Logs vs Traces**:
    - Metrics: Aggregated, numerical
    - Logs: Discrete events, text
    - Traces: Request flow across services

18. **OpenTelemetry Replaces**:
    - OpenTracing (deprecated)
    - OpenCensus (deprecated)

### CI/CD

19. **GitOps ≠ Git**:
    - Git is just the storage
    - Operator/agent does reconciliation

20. **Helm 2 vs 3**:
    - Helm 3 removed Tiller (security concern)
    - Most charts now v3

21. **Kustomize vs Helm**:
    - Kustomize: Template-free, overlays
    - Helm: Templates, package management

22. **Argo CD vs Flux**:
    - Argo: UI, pull-based
    - Flux: CLI-focused, also pull-based
    - Both are GitOps

### CNCF Landscape

23. **Envoy ≠ Istio**:
    - Envoy is proxy
    - Istio is service mesh (uses Envoy)

24. **containerd vs Docker**:
    - containerd is runtime
    - Docker uses containerd under hood

25. **CNI Plugin Required**:
    - K8s doesn't include networking
    - Must install CNI plugin

---

## 📝 Quick Reference Cheat Sheet

### Pod Lifecycle States
- Pending → Running → Succeeded/Failed
- CrashLoopBackOff: Container failing repeatedly

### Service Types Quick Pick
- Internal only? **ClusterIP**
- Need external access, static port? **NodePort**
- Need cloud load balancer? **LoadBalancer**
- Mapping to external DNS? **ExternalName**

### When to Use What Workload
| Use Case | Resource |
|----------|----------|
| Stateless app, scaling | Deployment |
| Database, ordered deployment | StatefulSet |
| One per node (logging) | DaemonSet |
| One-off task | Job |
| Scheduled task | CronJob |

### RBAC Quick Rules
- User/Group → ClusterRoleBinding → ClusterRole (cluster-wide)
- User/Group → RoleBinding → Role (namespace-scoped)
- ServiceAccount → For Pods, not humans

### Prometheus Metric Types
- Counter: Always up (requests_total)
- Gauge: Up or down (memory_usage)
- Histogram: Buckets (request_duration)
- Summary: Quantiles (request_duration)

### GitOps Core Principle
Git = Single source of truth → Automated sync → Declarative

### 12-Factor Speed Check
1. One codebase
2. Explicit dependencies
3. Config in env
4. Backing services as resources
5. Separate build/run
6. Stateless processes
7. Port binding
8. Scale by process
9. Fast startup/shutdown
10. Dev = Prod
11. Logs as streams
12. Admin tasks

---

## 🎯 Practice Question Patterns

### Pattern 1: CNCF Project Selection
**Q**: You need to implement service-to-service authentication and traffic encryption. Which CNCF project would you use?  
**A**: Istio or Linkerd (service mesh)

### Pattern 2: Architecture Decision
**Q**: An application needs to scale horizontally based on queue length. What should you use?  
**A**: KEDA (Kubernetes Event-Driven Autoscaling) + HPA

### Pattern 3: Resource Selection
**Q**: You need to run a log collection agent on every node in your cluster. What resource type?  
**A**: DaemonSet

### Pattern 4: Troubleshooting
**Q**: A Pod is in Pending state. What should you check first?  
**A**: kubectl describe pod (check Events for scheduling issues)

### Pattern 5: Security
**Q**: How do you restrict which Pods can communicate with a database Pod?  
**A**: NetworkPolicy

### Pattern 6: Storage
**Q**: You need persistent storage that survives Pod deletion. What do you use?  
**A**: PersistentVolumeClaim (PVC) bound to PersistentVolume (PV)

### Pattern 7: Observability
**Q**: You need to visualize metrics from multiple services. Which tools?  
**A**: Prometheus (collect) + Grafana (visualize)

### Pattern 8: Configuration
**Q**: How do you inject non-sensitive configuration into containers?  
**A**: ConfigMap (as env vars or volume mount)

---

## 🚨 Common Mistakes to Avoid

1. **Confusing Docker with containerd**: Docker is not K8s runtime anymore
2. **Thinking Helm requires Tiller**: Helm 3 removed it
3. **Assuming RBAC denies by default**: It's permissive by default
4. **Mixing up metrics-server and kube-state-metrics**: Different purposes
5. **Forgetting CNI plugins**: K8s needs one for networking
6. **Thinking service mesh is always needed**: Adds complexity, not always worth it
7. **Assuming Prometheus stores long-term**: It doesn't (use Thanos)
8. **Confusing Pod phase with container state**: Different concepts
9. **Thinking GitOps = CI**: GitOps is CD (continuous deployment)
10. **Forgetting OpenTelemetry replaced OpenTracing**: Use OTel for new projects

---

## 🎓 Final 24 Hours Checklist

### Must Review:
- [ ] All 5 exam domains weights and topics
- [ ] Kubernetes architecture diagram (control plane + node components)
- [ ] All workload resource types and when to use each
- [ ] Service types and differences
- [ ] RBAC components and relationships
- [ ] CNCF graduated projects (know top 10 cold)
- [ ] Prometheus, Fluentd, Jaeger - what each does
- [ ] GitOps principles
- [ ] 12-factor app methodology
- [ ] Service mesh purpose and examples
- [ ] Helm vs Kustomize
- [ ] Argo CD vs Flux

### Quick Drills:
- [ ] CNCF project → problem mapping (20 questions)
- [ ] Workload resource selection quiz (15 questions)
- [ ] Service type scenarios (10 questions)
- [ ] Security concepts (RBAC, NetworkPolicy, Pod security)
- [ ] Observability stack components

### Mental Preparation:
- [ ] Get 8 hours sleep before exam
- [ ] Exam is 60 questions, 90 minutes = 1.5 min per question
- [ ] Skip hard questions, come back later
- [ ] Trust your gut on conceptual questions
- [ ] 75% to pass, but aim for 90%+

---

## 📚 Essential Resources

### Official:
- **CNCF KCNA Curriculum**: https://github.com/cncf/curriculum
- **Kubernetes Docs**: https://kubernetes.io/docs/
- **CNCF Landscape**: https://landscape.cncf.io/

### Courses:
- KodeKloud KCNA course (highly recommended)
- Linux Foundation LFS250 course

### Practice Exams:
- KodeKloud Practice Tests
- Udemy Practice Exams (multiple vendors)

### Hands-on (Optional but helpful):
- Killercoda K8s playground
- Minikube or Kind locally
- Play with kubectl commands

---

## 💪 You've Got This!

You're already mid-CKA level, which means:
- ✅ You know K8s architecture cold
- ✅ You understand all the core resources
- ✅ You grasp orchestration concepts

What you need to add:
- 🎯 Broader CNCF ecosystem knowledge
- 🎯 Cloud-native architecture patterns
- 🎯 Non-K8s CNCF projects (Prometheus, Fluentd, etc.)
- 🎯 CI/CD and GitOps concepts

**This is conceptual validation of what you already know.**

With 2-3 days of focused study on this guide, you'll crush it.

**Target**: 90%+ score (definitely achievable)

Now go make it happen! 🚀

---

## 🎯 Post-Exam: Next Steps

After passing KCNA:
1. **CKA** (Certified Kubernetes Administrator) - your next cert
2. **CKAD** (Certified Kubernetes Application Developer)
3. **CKS** (Certified Kubernetes Security Specialist)
4. **KCSA** (Kubernetes & Cloud Native Security Associate)

Then explore CNCF project-specific certs:
- Prometheus (PCA)
- Istio (ICA)
- Cilium (CCA)
- Argo (CAPA)

**Kubestronaut** = All 5 K8s certs (KCNA, KCSA, CKA, CKAD, CKS)  
**Golden Kubestronaut** = Kubestronaut + 10 more CNCF certs

You're on your way! 💪

---

*Last Updated: December 2025*  
*Version: 1.0*  
*Target Exam: KCNA 2025*
