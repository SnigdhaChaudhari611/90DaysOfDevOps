# Day 50 - Kubernetes Architecture and Cluster Setup

## 1. Kubernetes Story

### Why was Kubernetes created?

Docker makes it easy to build and run containers, but managing a large number of containers across multiple servers becomes difficult manually.

Kubernetes is a **container orchestration platform** that helps automate the deployment, scheduling, scaling, networking, and recovery of containerized workloads.

A useful way to remember it:

> **Containers are ephemeral. Kubernetes continuously works to maintain the desired state.**

### Who created Kubernetes?

Kubernetes was created at **Google** and was heavily influenced by Google's internal container-management system called **Borg**.

Borg gave Google experience managing large-scale workloads and containers, and many of those lessons influenced Kubernetes.

### What does Kubernetes mean?

Kubernetes comes from a Greek word meaning **helmsman** or **captain/pilot of a ship**.

---

# 2. Kubernetes Architecture

## The simple mental model

Think of a Kubernetes cluster like a company.

### Control Plane = Management

| Kubernetes Component | Easy way to remember | Main job |
|---|---|---|
| API Server | Front door / Team Lead | Receives and processes Kubernetes API requests |
| etcd | Company database | Stores persistent cluster state |
| Scheduler | HR | Decides which suitable worker node should run a new Pod |
| Controller Manager | Project Manager | Continuously works to keep actual state aligned with desired state |

### Data Plane = Work

| Kubernetes Component | Easy way to remember | Main job |
|---|---|---|
| Worker Node | Employee workstation | Runs application workloads |
| kubelet | Node agent / Local manager | Makes sure Pods assigned to the node are running |
| Container Runtime | The worker that actually runs containers | Runs containers, such as through containerd or CRI-O |
| kube-proxy | Service traffic director | Helps implement Kubernetes Service networking |

### Core memory trick

> **Control Plane decides.**  
> **Data Plane runs.**

---

## `kubectl apply` Flow

When I run:

```bash
kubectl apply -f pod.yaml
```

the simplified flow is:

```text
You
 ↓
kubectl
 ↓
API Server
 ↓
etcd
 ↓
Scheduler
 ↓
API Server
 ↓
kubelet on selected Worker Node
 ↓
Container Runtime
 ↓
Pod
```

### What happens?

1. `kubectl` sends the request to the API Server.
2. The API Server validates and processes the request.
3. The Pod object is stored in etcd.
4. The Scheduler notices that the Pod needs a node.
5. The Scheduler selects a suitable worker node.
6. The assignment is recorded through the API Server.
7. The kubelet on the selected node notices the Pod assignment.
8. kubelet works with the container runtime to start the required containers.
9. The Pod starts running.

### Important distinction

> **Scheduler decides WHERE the Pod runs.**  
> **kubelet makes sure the Pod runs THERE.**

`kube-proxy` is not a required step in the basic Pod creation flow. It mainly helps implement Service networking.

---

# 3. Failure Scenarios

## What happens if the API Server goes down?

The API Server is a critical part of the control plane, but an API Server outage does **not automatically stop all already-running containers**.

Existing workloads on healthy worker nodes can continue running.

However, Kubernetes cannot properly process new API requests, and normal scheduling and controller reconciliation are disrupted until the API Server becomes available again.

Simple idea:

> **API Server down = Kubernetes management is severely affected, but existing workloads do not necessarily stop immediately.**

---

## What happens if a Worker Node goes down?

Kubernetes detects that the node is unhealthy through node status/heartbeats.

The Node Controller, which is part of the Controller Manager, can mark the node as unhealthy/`NotReady`.

If the workload is managed by a controller such as a Deployment, Kubernetes can work toward recreating replacement Pods on healthy nodes.

Important:

> A standalone Pod is not automatically recreated elsewhere in the same way a Deployment-managed Pod is.

---

# 4. Local Cluster Setup

## Tool chosen: kind

I chose **kind (Kubernetes in Docker)**.

### Why kind?

I already have experience with Docker, and kind uses Docker containers as Kubernetes nodes. This makes it easy to create, experiment with, delete, and recreate local Kubernetes clusters.

> **Important:** kind and minikube are both mainly local tools for learning, development, testing, and CI. The choice is not because minikube is only for development while kind is for staging.

### Install kubectl

macOS:

```bash
brew install kubectl
```

Verify:

```bash
kubectl version --client
```

### Install kind

```bash
brew install kind
```

### Create the cluster

```bash
kind create cluster --name devops-cluster
```

### Verify

```bash
kubectl cluster-info
kubectl get nodes
```

Expected result:

```text
NAME                        STATUS   ROLES           AGE   VERSION
devops-cluster-control-plane   Ready    control-plane   ...   ...
```

### Docker + kind connection

kind creates Kubernetes nodes as Docker containers.

```text
Docker
  ↓
kind
  ↓
Kubernetes Node
  ↓
Kubernetes Pods
```

---

# 5. Explore the Cluster

### Cluster information

```bash
kubectl cluster-info
```

### List nodes

```bash
kubectl get nodes
```

### Detailed node information

```bash
kubectl describe node <node-name>
```

### List namespaces

```bash
kubectl get namespaces
```

### List all Pods in all namespaces

```bash
kubectl get pods -A
```

### Look at the kube-system namespace

```bash
kubectl get pods -n kube-system
```

---

# 6. kube-system Pods

The exact list can vary depending on the cluster setup, but a kind cluster commonly includes components such as:

| Pod / Component | Purpose |
|---|---|
| `etcd` | Stores Kubernetes cluster state |
| `kube-apiserver` | Kubernetes API entry point |
| `kube-scheduler` | Assigns unscheduled Pods to suitable nodes |
| `kube-controller-manager` | Runs controllers that reconcile cluster state |
| `kube-proxy` | Helps implement Service networking |
| `coredns` | Provides DNS-based service discovery inside the cluster |

### Important

Some Kubernetes components run as Pods, while others are provided differently depending on the cluster/distribution.

For a local kind cluster, many control-plane components are visible as Pods in `kube-system`.

---

# 7. Cluster Lifecycle

## Delete the cluster

```bash
kind delete cluster --name devops-cluster
```

## Recreate it

```bash
kind create cluster --name devops-cluster
```

## Verify

```bash
kubectl get nodes
```

---

# 8. Kubernetes Contexts and kubeconfig

When we use `kubectl`, it needs to know **which Kubernetes cluster we want to talk to** and **how to connect to it**.

This information is stored in a file called **kubeconfig**.

The default kubeconfig location is:

```text
~/.kube/config
```

Think of kubeconfig as a **connection/configuration file for kubectl**.

It can contain information about multiple Kubernetes clusters.

For example, I might have:

```text
Local kind cluster
Staging cluster
Production cluster
```

I don't want to manually enter the API Server address and authentication details every time I use `kubectl`.

The kubeconfig keeps this information for me.

## Three important terms

### 1. Cluster

A **cluster** is the Kubernetes environment itself.

For example:

```text
devops-cluster
```

The kubeconfig stores information about how to connect to that cluster, including its API Server.

Think:

> **Cluster = Which Kubernetes environment am I connecting to?**

---

### 2. User

A **user** represents the identity/credentials used to access the cluster.

Think:

> **User = Who am I when I connect to this Kubernetes cluster?**

---

### 3. Context

A **context** connects a cluster and a user together.

It tells `kubectl`:

> **"For this session, use THIS cluster with THIS user."**

For example:

```text
Context: devops-context
    ↓
Cluster: devops-cluster
    ↓
User: my-user
```

This becomes especially useful when you have multiple clusters.

For example:

```text
Context: local
    → local kind cluster

Context: staging
    → staging cluster

Context: production
    → production cluster
```

Then instead of manually specifying everything, I can switch the context and `kubectl` knows which cluster my commands should go to.

## Useful commands

### Check which context kubectl is currently using

```bash
kubectl config current-context
```

This answers:

> "Which cluster is my kubectl currently pointing toward?"

### List all available contexts

```bash
kubectl config get-contexts
```

This shows the contexts available in your kubeconfig.

### Switch to another context

```bash
kubectl config use-context <context-name>
```

This changes which cluster `kubectl` will use for subsequent commands.

### View the kubeconfig

```bash
kubectl config view
```

This displays the configuration that `kubectl` is using.

## Easy way to remember

```text
kubeconfig
    ↓
stores connection information
    ↓
 ┌──────────┐
 │ Cluster  │ → Which Kubernetes cluster?
 └──────────┘
      +
 ┌──────────┐
 │   User   │ → Who is connecting?
 └──────────┘
      +
 ┌──────────┐
 │ Context  │ → Which cluster + which user?
 └──────────┘
      ↓
   kubectl
      ↓
Kubernetes API Server
```

### Important distinction

`kubectl config` does **not** configure the Kubernetes cluster itself.

It manages the configuration that **your local `kubectl` uses to connect to Kubernetes clusters**.

For the current kind setup, I don't need to worry about managing multiple contexts yet. The important thing to understand is **why kubeconfig and contexts exist**.

---

# 9. Screenshots

## `kubectl get nodes`

_Add screenshot here:_

```text
<!-- Screenshot: kubectl get nodes -->
```

## `kubectl get pods -n kube-system`

_Add screenshot here:_

```text
<!-- Screenshot: kubectl get pods -n kube-system -->
```

---

# 10. Quick Revision

```text
Kubernetes Cluster
│
├── Control Plane
│   ├── API Server       → Front door
│   ├── etcd             → Cluster database
│   ├── Scheduler        → Chooses node for Pod
│   └── Controller Mgr   → Reconciles desired vs actual state
│
└── Data Plane
    └── Worker Nodes
        ├── kubelet      → Manages Pods on the node
        ├── kube-proxy   → Service networking
        ├── Runtime      → Runs containers
        └── Pods         → Application workloads
```

### The easiest way to remember Kubernetes

> **Control Plane decides.**  
> **Worker Nodes run.**  
> **API Server is the front door.**  
> **Scheduler chooses the node.**  
> **kubelet makes the Pod run.**  
> **etcd remembers the state.**  
> **Controllers keep the desired state in place.**
