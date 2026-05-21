# Kubernetes Architecture Deep Dive

## Table of Contents
- [What is Kubernetes?](#what-is-kubernetes)
- [Kubernetes Architecture Overview](#kubernetes-architecture-overview)
- [Control Plane Components](#control-plane-components)
- [Worker Node Components](#worker-node-components)
- [How They Work Together](#how-they-work-together)
- [Application Deployment Flow](#application-deployment-flow)
- [Communication Flow](#communication-flow)

---

## What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. Originally developed by Google and now maintained by the Cloud Native Computing Foundation (CNCF), Kubernetes provides a robust framework for running distributed systems resiliently.

### Key Capabilities:
- **Container Orchestration**: Manages containers across multiple hosts
- **Self-Healing**: Automatically restarts failed containers and reschedules them
- **Auto-Scaling**: Scales applications based on resource utilization
- **Load Balancing**: Distributes network traffic across containers
- **Rolling Updates**: Enables zero-downtime deployments
- **Service Discovery**: Automatically assigns DNS names and IPs to services
- **Storage Orchestration**: Mounts storage systems of your choice
- **Secret & Configuration Management**: Manages sensitive information securely

---

## Kubernetes Architecture Overview

Kubernetes follows a **master-worker architecture** (also called control plane-worker node architecture). The cluster consists of:

1. **Control Plane**: The brain of the cluster that manages the cluster state
2. **Worker Nodes**: Machines that run your containerized applications

```
┌─────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                            │
│  ┌──────────────┐  ┌──────┐  ┌───────────┐  ┌────────────┐ │
│  │  API Server  │  │ etcd │  │ Scheduler │  │ Controller │ │
│  │              │  │      │  │           │  │  Manager   │ │
│  └──────────────┘  └──────┘  └───────────┘  └────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ (Control)
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼────────┐    ┌───────▼────────┐    ┌──────▼─────────┐
│  Worker Node 1 │    │  Worker Node 2 │    │  Worker Node 3 │
│  ┌──────────┐  │    │  ┌──────────┐  │    │  ┌──────────┐  │
│  │ kubelet  │  │    │  │ kubelet  │  │    │  │ kubelet  │  │
│  │kube-proxy│  │    │  │kube-proxy│  │    │  │kube-proxy│  │
│  │Container │  │    │  │Container │  │    │  │Container │  │
│  │ Runtime  │  │    │  │ Runtime  │  │    │  │ Runtime  │  │
│  └──────────┘  │    │  └──────────┘  │    │  └──────────┘  │
│   [PODs]       │    │   [PODs]       │    │   [PODs]       │
└────────────────┘    └────────────────┘    └────────────────┘
```

---

## Control Plane Components

The Control Plane makes global decisions about the cluster (like scheduling) and detects and responds to cluster events. It can run on any machine in the cluster, but typically runs on dedicated master nodes for high availability.

### 1. **kube-apiserver**

**Role**: The front-end of the Kubernetes control plane and the primary management point for the cluster.

**Detailed Functions**:
- **RESTful API Gateway**: Exposes Kubernetes API that all other components interact with
- **Authentication & Authorization**: Validates and authorizes all API requests
- **Admission Control**: Enforces policies and mutates/validates objects before persistence
- **API Request Handler**: Processes REST operations (GET, POST, PUT, DELETE, PATCH)
- **Only Component that Talks to etcd**: All cluster state modifications go through API server

**How it Works**:
```
kubectl → API Server → Authentication → Authorization → Admission Controllers → etcd
```

**Key Characteristics**:
- Stateless and horizontally scalable
- Listens on port 6443 (HTTPS) by default
- All component communication flows through API server (hub-and-spoke model)
- Implements watch mechanism for real-time updates

**Example Interaction**:
```bash
# When you run:
kubectl create deployment nginx --image=nginx

# The API server:
# 1. Authenticates your request
# 2. Authorizes based on RBAC rules
# 3. Validates the deployment specification
# 4. Stores the deployment spec in etcd
# 5. Notifies watchers (scheduler, controller manager)
```

---

### 2. **etcd**

**Role**: Distributed key-value store that serves as Kubernetes' backing store for all cluster data.

**Detailed Functions**:
- **Cluster State Database**: Stores the entire cluster state and configuration
- **Source of Truth**: The only persistent storage in the cluster
- **Configuration Storage**: Stores all resource definitions (Pods, Services, ConfigMaps, etc.)
- **Distributed Consensus**: Uses Raft consensus algorithm for consistency

**What's Stored in etcd**:
- Cluster configuration and metadata
- Resource specifications (Deployments, Services, ConfigMaps, Secrets)
- Current state of all resources
- Network policies and RBAC rules
- Node information and health status

**Key Characteristics**:
- Strongly consistent and highly available
- Requires odd number of nodes (3, 5, 7) for quorum
- Listens on port 2379 for client requests
- Port 2380 for peer communication
- Regular backups are critical (disaster recovery)

**Data Structure Example**:
```
/registry/
├── pods/
│   ├── default/
│   │   ├── nginx-pod-1
│   │   └── nginx-pod-2
├── services/
│   └── default/
│       └── nginx-service
├── deployments/
│   └── default/
│       └── nginx-deployment
└── nodes/
    ├── node-1
    └── node-2
```

**Performance Considerations**:
- Use SSD storage for better I/O performance
- Keep cluster size reasonable (recommended: < 5000 nodes)
- Monitor etcd health and latency

---

### 3. **kube-scheduler**

**Role**: Watches for newly created Pods with no assigned node and selects a node for them to run on.

**Detailed Functions**:
- **Pod Placement**: Decides which node should run each Pod
- **Resource Optimization**: Balances resource utilization across nodes
- **Constraint Evaluation**: Considers requirements, policies, and affinities
- **Scoring Mechanism**: Ranks suitable nodes and picks the best one

**Scheduling Process**:

**Phase 1 - Filtering** (Find feasible nodes):
- **Resource Requirements**: Check CPU, memory, storage requests
- **Node Selector**: Match labels specified in Pod spec
- **Node Affinity/Anti-Affinity**: Evaluate node preference rules
- **Pod Affinity/Anti-Affinity**: Consider Pod-to-Pod relationships
- **Taints and Tolerations**: Check if Pod can tolerate node taints
- **Volume Requirements**: Verify storage availability

**Phase 2 - Scoring** (Rank feasible nodes):
- **Least Requested Priority**: Prefer nodes with more available resources
- **Balanced Resource Allocation**: Distribute CPU and memory evenly
- **Image Locality**: Prefer nodes that already have container images
- **Inter-Pod Affinity**: Consider Pod topology preferences

**Example Scheduling Decision**:
```yaml
# Pod with specific requirements
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    disktype: ssd
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
        topologyKey: kubernetes.io/hostname
```

**Scheduler Workflow**:
```
1. Watch API server for unscheduled Pods (nodeName == "")
2. Filter nodes based on constraints
3. Score remaining nodes
4. Select highest-scoring node
5. Bind Pod to node via API server
6. API server updates Pod spec with nodeName
7. kubelet on selected node creates containers
```

---

### 4. **kube-controller-manager**

**Role**: Runs controller processes that regulate the state of the cluster and make changes to move the current state toward the desired state.

**Detailed Functions**:
- **State Reconciliation**: Continuously monitors and corrects cluster state
- **Multiple Controllers**: Runs various controllers as separate processes
- **Watch-Act Loop**: Watches API server for changes and takes corrective actions

**Core Controllers**:

**a) Node Controller**:
- Monitors node health and status
- Detects node failures and marks them as unhealthy
- Evicts Pods from failed nodes after grace period (default: 5 minutes)
- Updates node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)

**b) Replication Controller / ReplicaSet Controller**:
- Ensures correct number of Pod replicas are running
- Creates new Pods if count is below desired state
- Deletes Pods if count exceeds desired state
- Handles Pod failures by creating replacements

**c) Deployment Controller**:
- Manages Deployment resources
- Handles rolling updates and rollbacks
- Creates and manages ReplicaSets
- Implements update strategies (RollingUpdate, Recreate)

**d) StatefulSet Controller**:
- Manages StatefulSets for stateful applications
- Provides stable network identities and persistent storage
- Maintains ordered Pod creation/deletion

**e) DaemonSet Controller**:
- Ensures specific Pods run on all (or selected) nodes
- Useful for node-level operations (logging, monitoring agents)

**f) Job Controller**:
- Manages Job resources for batch processing
- Creates Pods and tracks completion
- Handles retries and parallelism

**g) CronJob Controller**:
- Manages scheduled Jobs
- Creates Jobs based on cron schedules

**h) Service Account Controller**:
- Creates default ServiceAccounts for namespaces
- Manages tokens for ServiceAccounts

**i) Endpoint Controller**:
- Populates Endpoints objects (links Services to Pods)
- Updates endpoints when Pods are added/removed

**Controller Reconciliation Loop**:
```go
// Pseudo-code for controller logic
for {
    desiredState = getFromAPIServer()
    currentState = observeCluster()
    
    if currentState != desiredState {
        takeActionToReconcile()
    }
    
    sleep(syncPeriod)
}
```

**Example: ReplicaSet Controller in Action**:
```yaml
# Desired state: 3 replicas
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2

# If only 2 Pods exist:
# Controller creates 1 more Pod
# If 4 Pods exist:
# Controller deletes 1 Pod
```

---

### 5. **cloud-controller-manager** (Optional)

**Role**: Integrates Kubernetes with cloud provider-specific APIs (AWS, GCP, Azure).

**Detailed Functions**:
- **Node Controller**: Updates node information from cloud provider
- **Route Controller**: Configures routes in cloud network
- **Service Controller**: Creates/deletes cloud load balancers for LoadBalancer Services
- **Volume Controller**: Creates, attaches, and mounts cloud volumes

**Cloud-Specific Operations**:

**AWS Example**:
- Creates ELB/ALB for LoadBalancer Services
- Provisions EBS volumes for PersistentVolumes
- Manages security groups and IAM roles
- Updates node labels with availability zone info

**GCP Example**:
- Creates Google Cloud Load Balancer
- Provisions Persistent Disks
- Manages firewall rules

**Azure Example**:
- Creates Azure Load Balancer
- Provisions Azure Disks
- Manages network security groups

---

## Worker Node Components

Worker nodes are the machines that run containerized applications. Each node contains the necessary services to run Pods and is managed by the control plane.

### 1. **kubelet**

**Role**: The primary node agent that runs on each node and ensures containers are running in Pods.

**Detailed Functions**:
- **Pod Lifecycle Management**: Creates, starts, stops, and monitors containers
- **Node Registration**: Registers node with API server
- **Health Monitoring**: Reports node and Pod status to control plane
- **Resource Management**: Enforces resource limits and requests
- **Volume Management**: Mounts volumes into containers

**How kubelet Works**:

**a) Pod Specification Sources**:
- API server (primary source)
- Local file system (static Pods)
- HTTP endpoint

**b) Container Lifecycle**:
```
1. Watch API server for Pods assigned to this node
2. Download Pod specification
3. Pull container images
4. Create containers using container runtime
5. Start containers
6. Monitor container health (liveness/readiness probes)
7. Report status back to API server
8. Restart failed containers
9. Delete terminated Pods
```

**c) Health Checks**:

**Liveness Probe**: Determines if container should be restarted
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

**Readiness Probe**: Determines if container is ready to accept traffic
```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Startup Probe**: For slow-starting containers
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**d) Resource Enforcement**:
- Uses cgroups to limit CPU and memory
- Monitors resource usage
- Evicts Pods if node resources are exhausted
- Implements QoS classes (Guaranteed, Burstable, BestEffort)

**Key Characteristics**:
- Listens on port 10250
- Communicates with API server
- Talks to container runtime via CRI (Container Runtime Interface)
- Does NOT manage containers not created by Kubernetes

---

### 2. **kube-proxy**

**Role**: Network proxy that maintains network rules on nodes, enabling network communication to Pods.

**Detailed Functions**:
- **Service Abstraction**: Implements Kubernetes Service concept
- **Load Balancing**: Distributes traffic across Pod replicas
- **Network Rules Management**: Creates iptables/IPVS rules
- **Cluster Networking**: Enables Pod-to-Pod and external communication

**How kube-proxy Works**:

**a) Service Implementation Modes**:

**iptables Mode** (default):
- Creates iptables rules for each Service
- NAT-based load balancing
- Better performance than userspace mode
- Random load balancing across endpoints

**IPVS Mode**:
- Uses Linux IPVS (IP Virtual Server)
- Better performance for large clusters
- More load balancing algorithms (rr, lc, dh, sh, sed, nq)
- Requires IPVS kernel modules

**userspace Mode** (legacy):
- kube-proxy acts as a proxy
- Higher latency
- Rarely used now

**b) Service Types and kube-proxy**:

**ClusterIP** (default):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```
- kube-proxy creates rules to route traffic to Pod IPs
- Only accessible within cluster

**NodePort**:
```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```
- kube-proxy opens port on all nodes
- Routes traffic from nodePort to Pod IPs

**LoadBalancer**:
```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
```
- Creates external load balancer (cloud provider)
- kube-proxy routes traffic from LB to Pods

**ExternalName**:
```yaml
spec:
  type: ExternalName
  externalName: my.database.example.com
```
- DNS CNAME mapping
- No proxying involved

**c) Traffic Flow with kube-proxy**:
```
Client → Service IP:Port 
       ↓ (iptables/IPVS rule)
       → Pod IP:TargetPort (random selection among healthy endpoints)
```

**iptables Rules Example**:
```bash
# View Service rules
sudo iptables -t nat -L KUBE-SERVICES

# Chain for specific service
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-XXXXXX

# Load balancing across Pods
-A KUBE-SVC-XXXXXX -m statistic --mode random --probability 0.33 -j KUBE-SEP-POD1
-A KUBE-SVC-XXXXXX -m statistic --mode random --probability 0.50 -j KUBE-SEP-POD2
-A KUBE-SVC-XXXXXX -j KUBE-SEP-POD3
```

---

### 3. **Container Runtime**

**Role**: Software responsible for running containers on the node.

**Detailed Functions**:
- **Container Lifecycle**: Pull images, create, start, stop, delete containers
- **Image Management**: Store and manage container images
- **Resource Isolation**: Use namespaces and cgroups
- **Networking**: Set up container network interfaces

**Container Runtime Interface (CRI)**:
- Standard API for container runtimes
- kubelet uses CRI to communicate with runtime
- Allows pluggable runtimes

**Supported Runtimes**:

**a) containerd**:
- Industry-standard runtime
- CNCF graduated project
- Default in Kubernetes 1.24+
- Lightweight and efficient
- Used by Docker Desktop

**b) CRI-O**:
- Lightweight runtime specifically for Kubernetes
- OCI-compliant
- Optimized for Kubernetes workloads

**c) Docker Engine** (deprecated):
- Uses dockershim (removed in K8s 1.24)
- Must use cri-dockerd adapter now
- Still widely used but not recommended

**Runtime Comparison**:
```
┌────────────────────────────────────────────────────┐
│                    kubelet                          │
└────────────────────────────────────────────────────┘
                      │ CRI
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────▼────┐   ┌────▼─────┐  ┌───▼──────┐
   │containerd│   │  CRI-O   │  │cri-dockerd│
   └────┬────┘   └────┬─────┘  └───┬──────┘
        │             │             │
   ┌────▼────┐   ┌────▼─────┐  ┌───▼──────┐
   │  runc   │   │   runc   │  │ dockerd  │
   └─────────┘   └──────────┘  └──────────┘
```

**OCI (Open Container Initiative)**:
- Runtime Specification: How to run containers
- Image Specification: Container image format
- Distribution Specification: Image distribution

**Container Operations**:
```bash
# With containerd (via crictl)
crictl pull nginx:latest
crictl images
crictl ps
crictl logs <container-id>

# With CRI-O
crictl pull nginx:latest
crictl ps
```

---

## How They Work Together

### Complete Request Flow: Deploying an Application

Let's walk through what happens when you deploy an application to Kubernetes:

**Step 1: User Creates Deployment**
```bash
kubectl create deployment nginx --image=nginx:1.21 --replicas=3
```

**Step 2: API Server Processes Request**
- Authenticates user credentials
- Authorizes based on RBAC
- Validates Deployment specification
- Stores Deployment object in etcd
- Returns success to kubectl
- Notifies watchers (Controller Manager)

**Step 3: Deployment Controller Creates ReplicaSet**
- Watches for new Deployment objects
- Creates a ReplicaSet based on Deployment template
- Stores ReplicaSet in etcd via API server

**Step 4: ReplicaSet Controller Creates Pods**
- Watches for new ReplicaSet objects
- Creates 3 Pod objects (spec.replicas=3)
- Stores Pod objects in etcd with nodeName="" (unscheduled)

**Step 5: Scheduler Assigns Pods to Nodes**
- Watches for Pods with nodeName=""
- For each Pod:
  - Filters nodes (resources, affinity, taints)
  - Scores nodes (resource balance, locality)
  - Selects best node
  - Binds Pod to node (updates Pod.spec.nodeName in etcd)

**Step 6: kubelet Creates Containers**
- On each selected node, kubelet:
  - Watches API server for Pods assigned to this node
  - Sees new Pod assignment
  - Calls container runtime via CRI
  - Container runtime:
    - Pulls nginx:1.21 image
    - Creates container
    - Starts container
  - Monitors container health
  - Reports status to API server

**Step 7: kube-proxy Sets Up Networking**
- If Service is created:
```bash
kubectl expose deployment nginx --port=80 --type=ClusterIP
```
- kube-proxy on each node:
  - Watches for Service and Endpoint changes
  - Creates iptables/IPVS rules
  - Routes Service IP to Pod IPs

**Complete Flow Diagram**:
```
┌────────┐
│ kubectl│
└───┬────┘
    │ 1. kubectl create deployment
    ▼
┌────────────────────────────────────────────────┐
│         API Server (Control Plane)             │
│  • Authenticate & Authorize                    │
│  • Validate                                    │
│  • Store in etcd                               │
└───┬────────────────────────────────────────────┘
    │ 2. Store Deployment
    ▼
┌────────────────────────────────────────────────┐
│              etcd                              │
│  Deployment: nginx                             │
│  ReplicaSet: nginx-xxxxx                       │
│  Pods: nginx-xxxxx-pod1,pod2,pod3              │
└────────────────────────────────────────────────┘
    │ 3. Watch for changes
    ▼
┌────────────────────────────────────────────────┐
│      Controller Manager                        │
│  • Deployment Controller → Creates ReplicaSet  │
│  • ReplicaSet Controller → Creates Pods        │
└───┬────────────────────────────────────────────┘
    │ 4. Pods created (unscheduled)
    ▼
┌────────────────────────────────────────────────┐
│           Scheduler                            │
│  • Filter nodes                                │
│  • Score nodes                                 │
│  • Bind Pods to nodes                          │
└───┬────────────────────────────────────────────┘
    │ 5. Pods scheduled to nodes
    │
    ├──────────────────┬──────────────────┐
    ▼                  ▼                  ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node 1    │  │   Node 2    │  │   Node 3    │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │ kubelet │ │  │ │ kubelet │ │  │ │ kubelet │ │
│ └────┬────┘ │  │ └────┬────┘ │  │ └────┬────┘ │
│      │      │  │      │      │  │      │      │
│      │ 6. Create container    │      │      │
│      ▼      │  │      ▼      │  │      ▼      │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │Container│ │  │ │Container│ │  │ │Container│ │
│ │ Runtime │ │  │ │ Runtime │ │  │ │ Runtime │ │
│ └────┬────┘ │  │ └────┬────┘ │  │ └────┬────┘ │
│      │      │  │      │      │  │      │      │
│      ▼      │  │      ▼      │  │      ▼      │
│   [nginx]   │  │   [nginx]   │  │   [nginx]   │
│             │  │             │  │             │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │kube-proxy│ │  │ │kube-proxy│ │  │ │kube-proxy│ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
│ 7. Setup    │  │ 7. Setup    │  │ 7. Setup    │
│ iptables    │  │ iptables    │  │ iptables    │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

## Application Deployment Flow

### Example: Deploying a Complete Application Stack

**Application Architecture**:
- Frontend: React app (3 replicas)
- Backend: Node.js API (3 replicas)
- Database: PostgreSQL (1 replica, StatefulSet)
- Cache: Redis (1 replica)

**Step-by-Step Deployment**:

**1. Create Namespace**:
```bash
kubectl create namespace myapp
```
- API server creates namespace in etcd
- Namespace provides logical isolation

**2. Deploy PostgreSQL Database**:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres
  replicas: 1
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
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**Control Plane Actions**:
- StatefulSet controller creates Pod with stable identity
- Scheduler assigns Pod to node with available storage
- kubelet creates container
- PersistentVolume is attached to Pod
- Database initializes

**3. Deploy Backend API**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myapp/backend:v1.0
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          value: "postgres://postgres:5432/myapp"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 3000
```

**Control Plane Actions**:
- Deployment controller creates ReplicaSet
- ReplicaSet controller creates 3 Pods
- Scheduler distributes Pods across nodes for HA
- kubelets pull images and start containers
- Endpoint controller creates endpoints for Service
- kube-proxy creates routing rules

**4. Deploy Frontend**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: myapp/frontend:v1.0
        ports:
        - containerPort: 80
        env:
        - name: API_URL
          value: "http://backend-service"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: myapp
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

**Control Plane Actions**:
- Similar to backend deployment
- LoadBalancer Service triggers cloud-controller-manager
- Cloud provider creates external load balancer
- Load balancer routes external traffic to frontend Pods

**5. Application Flow**:
```
External User
      │
      ▼
Cloud Load Balancer (created by cloud-controller-manager)
      │
      ▼
kube-proxy rules (iptables/IPVS)
      │
      ▼
Frontend Pods (3 replicas)
      │ API calls
      ▼
backend-service (ClusterIP)
      │
      ▼
kube-proxy routes to Backend Pods
      │
      ▼
Backend Pods (3 replicas)
      │ Database queries
      ▼
postgres StatefulSet (1 replica)
      │
      ▼
PersistentVolume (storage)
```

---

## Communication Flow

### Control Plane ↔ Worker Node Communication

**1. kubelet → API Server**:
- Registers node on startup
- Sends heartbeats every 10 seconds (node status)
- Reports Pod status updates
- Reports resource usage metrics

**2. API Server → kubelet**:
- Sends Pod specifications for scheduling
- Requests Pod logs
- Executes commands in containers (kubectl exec)
- Port forwarding requests

**3. Controller Manager → API Server**:
- Watches for resource changes
- Creates/updates/deletes resources
- Updates resource status

**4. Scheduler → API Server**:
- Watches for unscheduled Pods
- Binds Pods to nodes

**5. kube-proxy → API Server**:
- Watches for Service and Endpoint changes
- Gets Service and Pod IP information

**Network Communication**:
```
┌─────────────────────────────────────────┐
│         Control Plane (Master)          │
│                                         │
│  API Server (port 6443)                 │
│      ▲         │                        │
│      │         │                        │
│      │         ▼                        │
│  Controller  Scheduler                  │
│  Manager                                │
│      ▲                                  │
│      │                                  │
│  etcd (port 2379)                       │
└─────────────────────────────────────────┘
       ▲         │
       │         │ HTTPS
       │         ▼
┌──────────────────────────────────────────┐
│          Worker Nodes                    │
│                                          │
│  kubelet (port 10250) ◄──► API Server   │
│     │                                    │
│     ▼                                    │
│  Container Runtime (CRI)                 │
│     │                                    │
│     ▼                                    │
│  Containers/Pods                         │
│                                          │
│  kube-proxy ◄──────────► API Server     │
│  (iptables/IPVS rules)                   │
└──────────────────────────────────────────┘
```

### Security Considerations

**Authentication**:
- Client certificates (kubectl)
- Bearer tokens (ServiceAccounts)
- Bootstrap tokens (kubeadm)
- OpenID Connect (enterprise SSO)

**Authorization**:
- RBAC (Role-Based Access Control)
- ABAC (Attribute-Based Access Control)
- Node authorization
- Webhook authorization

**Network Security**:
- TLS encryption for all control plane communication
- Network policies for Pod-to-Pod traffic
- Service mesh for advanced traffic management

**Secrets Management**:
- etcd encryption at rest
- Secret objects for sensitive data
- External secret management (HashiCorp Vault, AWS Secrets Manager)

---

## Summary

Kubernetes architecture is designed for resilience, scalability, and automation:

**Control Plane** (The Brain):
- **API Server**: Central hub for all communication
- **etcd**: Persistent storage for cluster state
- **Scheduler**: Intelligent Pod placement
- **Controller Manager**: State reconciliation
- **Cloud Controller Manager**: Cloud integration

**Worker Nodes** (The Muscle):
- **kubelet**: Pod lifecycle management
- **kube-proxy**: Network routing
- **Container Runtime**: Container execution

**Key Principles**:
1. **Declarative Configuration**: Define desired state, Kubernetes makes it happen
2. **Self-Healing**: Automatic recovery from failures
3. **Scalability**: Horizontal scaling of applications and cluster
4. **Modularity**: Pluggable components (CNI, CSI, CRI)
5. **API-Driven**: Everything via RESTful API

This architecture enables Kubernetes to manage thousands of containers across hundreds of nodes while maintaining high availability and reliability.

---

## Additional Resources

- Official Documentation: https://kubernetes.io/docs/
- Kubernetes Architecture: https://kubernetes.io/docs/concepts/architecture/
- kubectl Cheat Sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- CNCF Landscape: https://landscape.cncf.io/