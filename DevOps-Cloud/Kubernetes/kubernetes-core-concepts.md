# Kubernetes Core Concepts

Kubernetes (K8s) is a container orchestration platform that automates deployment, scaling, and management of
containerized applications. Interviews typically cover architecture, core workload resources, networking, storage,
and configuration management.

---

## Architecture

### Control Plane

The control plane manages the cluster state and makes scheduling decisions:

| Component              | Responsibility                                                      |
|------------------------|---------------------------------------------------------------------|
| `kube-apiserver`       | REST API gateway — the only component all others talk to            |
| `etcd`                 | Distributed key-value store — single source of truth for cluster state |
| `kube-scheduler`       | Assigns Pods to Nodes based on resource requirements and constraints |
| `kube-controller-manager` | Runs controllers (Deployment, ReplicaSet, Node, Job, etc.)      |
| `cloud-controller-manager` | Integrates with cloud provider APIs (load balancers, volumes, routes) |

### Node Components

Every worker node runs:

| Component        | Responsibility                                                         |
|------------------|------------------------------------------------------------------------|
| `kubelet`        | Agent that ensures containers described in PodSpecs are running        |
| `kube-proxy`     | Maintains network rules for Service traffic (iptables / IPVS)          |
| Container runtime| Runs containers — containerd (default), CRI-O                         |

### How a Deployment Happens

```
kubectl apply -f deployment.yaml
        │
        ▼
  kube-apiserver  ──▶  writes desired state to etcd
        │
        ▼
  controller-manager  ──▶  creates/updates ReplicaSet ──▶ creates Pod objects
        │
        ▼
  kube-scheduler  ──▶  assigns Pods to Nodes
        │
        ▼
  kubelet (on Node)  ──▶  pulls image, starts containers via container runtime
```

---

## Core Resources

### Pod

The **smallest deployable unit** — one or more containers that share:
- Network namespace (same IP, `localhost` communication).
- Storage volumes.
- Lifecycle (scheduled, started, and stopped together).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"      # 0.25 CPU core
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Pods are **ephemeral** — they are not rescheduled when they die. Use higher-level controllers (Deployment, StatefulSet)
to manage Pod lifecycle.

### Multi-Container Pod Patterns

| Pattern     | Use case                                        | Example                          |
|-------------|-------------------------------------------------|----------------------------------|
| Sidecar     | Helper that extends the main container          | Log collector, service mesh proxy |
| Init        | Runs to completion before main containers start | DB migration, config download    |
| Ambassador  | Proxy for outbound connections                  | Local proxy to a remote service  |
| Adapter     | Transforms output of the main container         | Log format converter             |

```yaml
spec:
  initContainers:
    - name: db-migrate
      image: myapp-migrate:1.0
      command: ["./migrate"]
  containers:
    - name: app
      image: myapp:1.0
    - name: log-agent
      image: fluentd:latest   # sidecar
```

---

## Workload Controllers

### ReplicaSet

Ensures a specified number of identical Pods are running at all times. Rarely used directly — managed by Deployments.

### Deployment

The standard way to manage **stateless** applications. Wraps a ReplicaSet and adds:
- Rolling updates and rollbacks.
- Version history.
- Scaling.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # max extra Pods during update
      maxUnavailable: 0   # zero-downtime update
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:2.0
          ports:
            - containerPort: 8080
```

**Rolling update flow:** new ReplicaSet is scaled up while old ReplicaSet is scaled down, one Pod at a time.

```bash
kubectl rollout status deployment/myapp    # watch progress
kubectl rollout history deployment/myapp   # view revisions
kubectl rollout undo deployment/myapp      # rollback to previous
kubectl rollout undo deployment/myapp --to-revision=3  # rollback to specific
```

### StatefulSet

For **stateful** applications (databases, message brokers). Guarantees:
- **Stable network identity** — Pods get predictable names (`mydb-0`, `mydb-1`, `mydb-2`).
- **Ordered deployment and scaling** — Pods are created/deleted sequentially.
- **Stable persistent storage** — each Pod gets its own PersistentVolumeClaim that survives rescheduling.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # headless Service for stable DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

### DaemonSet

Runs **one Pod per Node** (or a subset of Nodes). Use cases: log collectors, monitoring agents, network plugins,
storage daemons.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluentd:latest
```

### Job and CronJob

**Job** — runs Pods to completion (batch processing, migrations):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 3         # retries on failure
  ttlSecondsAfterFinished: 60
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp-migrate:1.0
```

**CronJob** — schedules Jobs on a cron expression:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"    # 2:00 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: report
              image: report-generator:1.0
```

### Workload Summary

| Controller   | Use case                    | Pod identity | Scaling order | Storage     |
|--------------|-----------------------------|--------------|---------------|-------------|
| Deployment   | Stateless apps              | Random       | Parallel      | Shared      |
| StatefulSet  | Stateful apps (DBs, queues) | Stable       | Sequential    | Per-Pod PVC |
| DaemonSet    | Node-level agents           | Per-Node     | —             | Host path   |
| Job          | Batch / one-off tasks       | Random       | Parallel      | Ephemeral   |
| CronJob      | Scheduled tasks             | Random       | Per schedule  | Ephemeral   |

---

## Networking

### Service

An abstraction that exposes a set of Pods as a stable network endpoint. Pods are selected by **labels**.

| Type           | Scope                   | How it works                                    |
|----------------|-------------------------|-------------------------------------------------|
| `ClusterIP`    | Internal only (default) | Virtual IP reachable only inside the cluster     |
| `NodePort`     | External via Node IP    | Exposes on a static port (30000–32767) on every Node |
| `LoadBalancer` | External via cloud LB   | Provisions a cloud load balancer → routes to NodePorts |
| `ExternalName` | DNS alias               | CNAME record pointing to an external service     |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp      # routes to Pods with this label
  ports:
    - port: 80        # Service port
      targetPort: 8080 # container port
```

**DNS:** within the cluster, Services are reachable at `<service>.<namespace>.svc.cluster.local`.

### Headless Service

A Service with `clusterIP: None` — no virtual IP is assigned. DNS returns **individual Pod IPs** directly. Used with
StatefulSets for stable per-Pod DNS (`postgres-0.postgres.default.svc.cluster.local`).

### Ingress

Layer 7 (HTTP/HTTPS) routing — maps hostnames and paths to backend Services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

Requires an **Ingress Controller** (NGINX, Traefik, AWS ALB, etc.) to be installed in the cluster.

### Gateway API

The successor to Ingress — more expressive, supports TCP/UDP, traffic splitting, and role-based configuration.
Defines `Gateway`, `HTTPRoute`, `GRPCRoute`, etc.

### Network Policies

Firewall rules at the Pod level — control which Pods can communicate with each other:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-only
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - port: 5432
```

By default, all Pods can communicate with all other Pods. NetworkPolicies are **additive** — if any policy selects a
Pod, only explicitly allowed traffic is permitted (deny by default for that Pod).

Requires a **CNI plugin** that supports Network Policies (Calico, Cilium, Weave Net).

---

## Configuration

### ConfigMap

Stores non-sensitive configuration data as key-value pairs:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

**Consuming a ConfigMap:**

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      # as environment variables
      envFrom:
        - configMapRef:
            name: app-config
      # or mount as files
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      configMap:
        name: app-config
```

### Secret

Like ConfigMap but for **sensitive data**. Values are base64-encoded (not encrypted by default — enable encryption at
rest via `EncryptionConfiguration`).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=      # base64("postgres")
  password: czNjcjN0cEBzcw==  # base64("s3cr3tp@ss")
```

Best practices:
- Enable encryption at rest for etcd.
- Use external secret managers (Vault, AWS Secrets Manager) with operators like External Secrets.
- Limit RBAC access to Secrets.
- Avoid committing Secrets to version control.

---

## Storage

### Volume Types

| Type                 | Lifecycle           | Use case                               |
|----------------------|---------------------|----------------------------------------|
| `emptyDir`           | Pod lifetime        | Scratch space, shared between containers in a Pod |
| `hostPath`           | Node lifetime       | Access host filesystem (DaemonSets)    |
| `persistentVolumeClaim` | Independent     | Persistent data (databases, files)     |
| `configMap` / `secret` | Resource lifetime | Mount config/secrets as files          |
| `projected`          | Pod lifetime        | Combine multiple sources into one mount |

### PersistentVolume (PV) and PersistentVolumeClaim (PVC)

**PV** — a piece of storage provisioned by an admin or dynamically via a StorageClass.
**PVC** — a request for storage by a user. Binds to a PV that satisfies the request.

```yaml
# PVC — request storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce       # RWO — single Node read/write
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

**Access modes:**

| Mode             | Abbreviation | Meaning                                    |
|------------------|-------------|--------------------------------------------|
| ReadWriteOnce    | RWO         | Single Node can mount read-write           |
| ReadOnlyMany     | ROX         | Multiple Nodes can mount read-only         |
| ReadWriteMany    | RWX         | Multiple Nodes can mount read-write        |
| ReadWriteOncePod | RWOP        | Single Pod can mount read-write (K8s 1.27+)|

### StorageClass

Defines **dynamic provisioning** — PVCs automatically create PVs via a provisioner:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete       # Delete or Retain when PVC is released
volumeBindingMode: WaitForFirstConsumer
```

---

## Resource Management

### Requests and Limits

| Field     | Meaning                                                        |
|-----------|----------------------------------------------------------------|
| `requests`| Guaranteed resources — scheduler uses this for placement       |
| `limits`  | Maximum resources — container is throttled (CPU) or OOMKilled (memory) if exceeded |

```yaml
resources:
  requests:
    cpu: "250m"       # 0.25 core — guaranteed
    memory: "256Mi"
  limits:
    cpu: "1"          # 1 core — throttled beyond this
    memory: "512Mi"   # OOMKilled if exceeded
```

**CPU units:** `1` = 1 vCPU/core. `100m` = 0.1 core (millicores).
**Memory units:** `Mi` (mebibytes), `Gi` (gibibytes).

### QoS Classes

Assigned automatically based on requests/limits:

| QoS Class    | Condition                                   | Eviction priority |
|--------------|---------------------------------------------|-------------------|
| Guaranteed   | requests == limits for all containers       | Last (lowest)     |
| Burstable    | At least one request set, limits differ     | Middle            |
| BestEffort   | No requests or limits set                   | First (highest)   |

### LimitRange and ResourceQuota

- **LimitRange** — default/max/min resource constraints per Pod/Container in a namespace.
- **ResourceQuota** — total resource budget for a namespace (max CPU, memory, Pod count, etc.).

### Horizontal Pod Autoscaler (HPA)

Automatically scales the number of replicas based on metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Vertical Pod Autoscaler (VPA)

Adjusts resource requests/limits of Pods based on historical usage. Useful when you don't know the right resource
values. Cannot be used together with HPA on the same metric.

---

## Scheduling

### Node Selection

```yaml
spec:
  # simple — match a label
  nodeSelector:
    disktype: ssd

  # flexible — required or preferred rules
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a", "us-east-1b"]
```

### Taints and Tolerations

Taints **repel** Pods from Nodes. Tolerations let specific Pods **ignore** a taint:

```bash
# Taint a node
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```

```yaml
# Pod tolerates the taint
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

| Effect           | Behavior                                               |
|------------------|--------------------------------------------------------|
| `NoSchedule`     | New Pods won't be scheduled unless they tolerate it    |
| `PreferNoSchedule` | Scheduler avoids the Node but may still place Pods  |
| `NoExecute`      | Existing non-tolerating Pods are evicted               |

### Pod Affinity / Anti-Affinity

Control Pod placement **relative to other Pods**:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname  # one replica per Node
```

---

## Health Checks (Probes)

| Probe       | Purpose                                                    | Failure action           |
|-------------|------------------------------------------------------------|--------------------------|
| `livenessProbe`  | Is the container alive?                               | Restart container        |
| `readinessProbe` | Is the container ready to receive traffic?            | Remove from Service endpoints |
| `startupProbe`   | Has the container finished starting? (slow-start apps)| Restart container        |

```yaml
containers:
  - name: app
    image: myapp:1.0
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 15
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 2    # 30 × 2s = 60s max startup time
```

**Probe types:** `httpGet`, `tcpSocket`, `exec` (run a command), `grpc`.

---

## Namespaces

Logical isolation within a cluster — not security boundaries by themselves.

```bash
kubectl get namespaces
# default, kube-system, kube-public, kube-node-lease
```

Use cases: team/environment separation, resource quotas, RBAC scoping, network policy boundaries.

Cross-namespace communication: `<service>.<namespace>.svc.cluster.local`.

---

## RBAC (Role-Based Access Control)

| Resource          | Scope      | What it defines                         |
|-------------------|------------|-----------------------------------------|
| `Role`            | Namespace  | Permissions within a single namespace   |
| `ClusterRole`     | Cluster    | Permissions across all namespaces       |
| `RoleBinding`     | Namespace  | Binds a Role/ClusterRole to a subject   |
| `ClusterRoleBinding` | Cluster | Binds a ClusterRole to a subject cluster-wide |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: read-pods
subjects:
  - kind: ServiceAccount
    name: monitoring
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ServiceAccounts** — identity for Pods. Each namespace has a `default` ServiceAccount. Create dedicated
ServiceAccounts with minimal permissions for each workload.

---

## Common Interview Questions

### What happens when you run `kubectl apply -f deployment.yaml`?

1. `kubectl` sends the manifest to `kube-apiserver` (REST API call).
2. API server validates, authenticates, applies admission controllers, and writes to `etcd`.
3. Deployment controller detects the new Deployment, creates/updates a ReplicaSet.
4. ReplicaSet controller creates Pod objects for the desired replica count.
5. Scheduler assigns each Pod to a Node (based on resources, affinity, taints).
6. `kubelet` on the assigned Node pulls the image and starts the container.
7. `kube-proxy` updates iptables/IPVS rules so the Service can route traffic to the new Pods.

### What is the difference between a Deployment and a StatefulSet?

- **Deployment** — stateless; Pods are interchangeable, get random names, share storage, scale in parallel.
- **StatefulSet** — stateful; Pods have stable identities (`app-0`, `app-1`), each gets its own PersistentVolumeClaim,
  and they are created/deleted sequentially.

### How does a Service route traffic to Pods?

A Service uses a **label selector** to find matching Pods. `kube-proxy` watches the Endpoints (or EndpointSlices) and
configures iptables/IPVS rules on each Node. When traffic hits the Service's ClusterIP, the kernel routes it to one
of the healthy backend Pod IPs using round-robin (iptables) or more advanced algorithms (IPVS).

### What is the difference between a liveness probe and a readiness probe?

- **Liveness** — "is this container stuck?" If it fails, the kubelet **restarts** the container.
- **Readiness** — "can this container handle traffic?" If it fails, the Pod is **removed from Service endpoints** but
  not restarted. It's re-added once the probe passes again.

A common mistake is using the same endpoint for both — a failing dependency makes liveness fail, causing a restart loop
instead of just removing traffic.

### How do you perform zero-downtime deployments?

1. Configure `readinessProbe` so new Pods only receive traffic when ready.
2. Use `RollingUpdate` strategy with `maxUnavailable: 0`.
3. Set proper `terminationGracePeriodSeconds` (default 30s) so in-flight requests complete.
4. Handle `SIGTERM` gracefully in your application — stop accepting new requests, drain existing ones.
5. Use `preStop` lifecycle hook if you need extra drain time:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]  # wait for endpoints to be updated
```

### What are taints and tolerations?

Taints on Nodes **repel** Pods that don't tolerate them. Tolerations on Pods allow them to be scheduled on tainted
Nodes. Use cases: dedicated GPU Nodes, prevent workloads on control plane Nodes, evict Pods from unhealthy Nodes.

### How does DNS work inside a Kubernetes cluster?

CoreDNS (default) runs as a Deployment in `kube-system`. Every Pod's `/etc/resolv.conf` points to the CoreDNS Service.
DNS records are created for:
- Services: `<service>.<namespace>.svc.cluster.local`
- Pods (if enabled): `<pod-ip-dashed>.<namespace>.pod.cluster.local`
- StatefulSet Pods: `<pod-name>.<headless-service>.<namespace>.svc.cluster.local`

### What is the difference between `kubectl apply` and `kubectl create`?

- `create` — imperative; fails if the resource already exists.
- `apply` — declarative; creates if not exists, updates if exists (three-way merge using the `last-applied-configuration` annotation).

Use `apply` in production — it supports GitOps workflows and tracks changes declaratively.
