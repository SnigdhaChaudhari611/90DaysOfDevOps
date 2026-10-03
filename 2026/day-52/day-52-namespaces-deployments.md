# Day 52 - Kubernetes Namespaces and Deployments

## Overview

Day 52 focused on **Kubernetes Namespaces and Deployments**.

The main goal was to understand:

- How Kubernetes namespaces organize resources
- How to run resources inside a specific namespace
- How a Deployment manages multiple Pod replicas
- How Deployments provide self-healing
- How to scale a Deployment
- How rolling updates and rollbacks work
- How to clean up Kubernetes resources

The official task expected at least two custom namespaces, a multi-replica Deployment, scaling, a rolling update, a rollback, and verification across namespaces.

---

# 1. Kubernetes Namespaces

A **Namespace** is a way to logically separate and organize resources inside the same Kubernetes cluster.

For example, different environments can use different namespaces:

```text
Kubernetes Cluster
│
├── default
├── dev
├── staging
├── production
└── kube-system
```

Namespaces are useful when multiple environments or teams share the same cluster.

For example:

```text
dev
└── application Pods

staging
└── application Pods

production
└── production application Pods
```

A resource can be targeted with:

```bash
kubectl get pods -n dev
```

or across all namespaces:

```bash
kubectl get pods -A
```

---

# 2. Explore Default Namespaces

First, I checked the namespaces already present in the cluster:

```bash
kubectl get namespaces
```

My cluster contained:

```text
default
kube-node-lease
kube-public
kube-system
local-path-storage
```

The important built-in namespaces are:

| Namespace | Purpose |
|---|---|
| `default` | Default namespace when no namespace is specified |
| `kube-system` | Kubernetes system components |
| `kube-public` | Resources that can be publicly readable |
| `kube-node-lease` | Node heartbeat/lease information |

I also checked the Pods running in `kube-system`:

```bash
kubectl get pods -n kube-system
```

There were **8 Pods** running there, including components such as:

- CoreDNS
- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kube-proxy
- kindnet

These are Kubernetes cluster components and should not be modified casually.

---

# 3. Create Custom Namespaces

I created two namespaces for development and staging:

```bash
kubectl create namespace dev
kubectl create namespace staging
```

Then verified them:

```bash
kubectl get namespaces
```

The namespaces appeared as:

```text
dev
staging
```

## Creating a Namespace using YAML

I also created a namespace using a manifest.

`namespace.yml`:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: production
```

Applied using:

```bash
kubectl apply -f namespace.yml
```

The result was:

```text
namespace/production created
```

### Errors encountered

I initially wrote:

```yaml
kind: namespace
apiversion: v1
```

This caused errors.

The correct Kubernetes field names and values are:

```yaml
kind: Namespace
apiVersion: v1
```

Kubernetes resource kinds and YAML field names must be written correctly.

---

# 4. Run Pods in Specific Namespaces

I created an Nginx Pod in the `dev` namespace:

```bash
kubectl run nginx-dev --image=nginx:latest -n dev
```

And another in `staging`:

```bash
kubectl run nginx-staging --image=nginx:latest -n staging
```

Then checked resources across all namespaces:

```bash
kubectl get pods -A
```

The Pods appeared under their respective namespaces:

```text
dev       nginx-dev
staging   nginx-staging
```

Running:

```bash
kubectl get pods
```

showed:

```text
No resources found in default namespace.
```

This demonstrates an important point:

> `kubectl get pods` checks the current/default namespace unless a namespace is specified.

To check a specific namespace:

```bash
kubectl get pods -n dev
```

To check everything:

```bash
kubectl get pods -A
```

---

# 5. Deployment vs Standalone Pod

A standalone Pod is an individual workload.

If the Pod is deleted, Kubernetes does not automatically recreate it just because it was originally created with:

```bash
kubectl run
```

A **Deployment** provides management of Pods through a desired replica count.

For example:

```yaml
replicas: 3
```

means:

> Kubernetes should maintain three Pods matching the Deployment's Pod template.

The basic relationship is:

```text
Deployment
    │
    ▼
ReplicaSet
    │
    ├── Pod
    ├── Pod
    └── Pod
```

The Deployment manages the ReplicaSet, and the ReplicaSet maintains the desired number of Pods.

---

# 6. Create the Nginx Deployment

I created:

`nginx-deployment.yaml`

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: nginx-deployment
  namespace: dev
  labels:
    app: nginx

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
          image: nginx:1.24
          ports:
            - containerPort: 80
```

## Explanation

### `apiVersion`

```yaml
apiVersion: apps/v1
```

Deployments belong to the `apps` API group.

### `kind`

```yaml
kind: Deployment
```

This tells Kubernetes that the object is a Deployment.

### `metadata`

```yaml
metadata:
  name: nginx-deployment
  namespace: dev
```

This gives the Deployment its name and places it inside the `dev` namespace.

### `replicas`

```yaml
replicas: 3
```

The Deployment should maintain three Pods.

### `selector`

```yaml
selector:
  matchLabels:
    app: nginx
```

The selector tells the Deployment which Pods belong to it.

### Pod template

```yaml
template:
  metadata:
    labels:
      app: nginx
```

This is the blueprint used to create the Pods.

The labels here must match the Deployment selector:

```yaml
selector:
  matchLabels:
    app: nginx
```

and:

```yaml
template:
  metadata:
    labels:
      app: nginx
```

### Container

```yaml
containers:
  - name: nginx
    image: nginx:1.24
```

The Pods run the `nginx:1.24` container image.

---

# 7. YAML Indentation Error

My first Deployment manifest had the Pod `spec` incorrectly placed inside `metadata`.

The error was:

```text
unknown field "spec.template.metadata.spec"
```

The incorrect structure was effectively:

```yaml
template:
  metadata:
    labels:
      app: nginx
    spec:
```

The correct structure is:

```yaml
template:
  metadata:
    labels:
      app: nginx

  spec:
    containers:
```

The important structure to remember is:

```text
Deployment
└── spec
    ├── replicas
    ├── selector
    └── template
        ├── metadata
        │   └── labels
        └── spec
            └── containers
```

After fixing the indentation:

```bash
kubectl apply -f nginx-deployment.yaml
```

returned:

```text
deployment.apps/nginx-deployment created
```

---

# 8. Verify the Deployment

I checked the Deployment:

```bash
kubectl get deployments -n dev
```

Output:

```text
NAME               READY   UP-TO-DATE   AVAILABLE
nginx-deployment   3/3     3            3
```

### Meaning of the columns

| Column | Meaning |
|---|---|
| READY | Number of ready replicas / desired replicas |
| UP-TO-DATE | Pods using the latest Deployment configuration |
| AVAILABLE | Pods currently available to serve the workload |

So:

```text
3/3
```

means all three desired replicas are ready.

I also checked the Pods:

```bash
kubectl get pods -n dev
```

There were three Deployment-managed Pods:

```text
nginx-deployment-7f5f95d8d-5xhfc
nginx-deployment-7f5f95d8d-lcrgw
nginx-deployment-7f5f95d8d-n8q6m
```

---

# 9. Self-Healing

This was one of the most important parts of the exercise.

I deleted a Deployment-managed Pod:

```bash
kubectl delete pod nginx-deployment-7f5f95d8d-5xhfc -n dev
```

Kubernetes immediately created another Pod.

Before:

```text
nginx-deployment-7f5f95d8d-5xhfc
nginx-deployment-7f5f95d8d-lcrgw
nginx-deployment-7f5f95d8d-n8q6m
```

After deletion:

```text
nginx-deployment-7f5f95d8d-f99nl
nginx-deployment-7f5f95d8d-lcrgw
nginx-deployment-7f5f95d8d-n8q6m
```

The replacement Pod had a **different name**.

I repeated this with the other Pods and observed the same behavior.

### Why did Kubernetes recreate it?

The Deployment's desired state was:

```text
3 replicas
```

After deleting one Pod:

```text
Desired: 3
Current: 2
```

The ReplicaSet noticed the difference and created another Pod:

```text
Desired: 3
Current: 3
```

This is Kubernetes' **self-healing behavior**.

---

# 10. Scaling the Deployment

I used the imperative scaling command:

```bash
kubectl scale deployment/nginx-deployment -n dev --replicas=4
```

Kubernetes created another Pod so that the Deployment had four replicas.

I then scaled down:

```bash
kubectl scale deployment/nginx-deployment -n dev --replicas=1
```

Only one Deployment Pod remained.

I also tested scaling up to ten replicas:

```bash
kubectl scale deployment/nginx-deployment -n dev --replicas=10
```

Kubernetes created enough Pods to reach ten replicas.

Then I scaled it back down:

```bash
kubectl scale deployment/nginx-deployment -n dev --replicas=1
```

Only one Deployment Pod remained.

## Imperative vs Declarative Scaling

### Imperative

Tell Kubernetes directly:

```bash
kubectl scale deployment/nginx-deployment --replicas=4 -n dev
```

### Declarative

Change the YAML:

```yaml
spec:
  replicas: 4
```

Then apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

The key idea is:

```text
Desired state
     ↓
Deployment
     ↓
Kubernetes controllers
     ↓
Actual state
```

Kubernetes continuously works to make the actual state match the desired state.

---

# 11. Rolling Update

I changed the Nginx image using:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev
```

Then checked the rollout:

```bash
kubectl rollout status deployment/nginx-deployment -n dev
```

Output:

```text
deployment "nginx-deployment" successfully rolled out
```

This created a new Deployment revision.

The general rolling update idea is:

```text
Old Pods
   ↓
New Pods created
   ↓
New Pods become ready
   ↓
Old Pods are gradually replaced
```

A rolling update avoids replacing every Pod at exactly the same time.

> Note: A rolling update supports high availability, but saying it automatically guarantees "zero downtime" is too broad. Actual availability depends on factors such as readiness, replicas, update strategy, and application behavior.

---

# 12. Rollout History

I checked the Deployment history:

```bash
kubectl rollout history deployment/nginx-deployment -n dev
```

The Deployment showed multiple revisions:

```text
REVISION
2
3
```

Each new Deployment configuration that results in a new ReplicaSet can create a new revision.

---

# 13. Rollback

I tested rolling back the Deployment:

```bash
kubectl rollout undo deployment/nginx-deployment -n dev
```

Then checked the rollout:

```bash
kubectl rollout status deployment/nginx-deployment -n dev
```

The rollout completed successfully.

I verified the running image using:

```bash
kubectl describe deployment nginx-deployment -n dev | grep Image
```

The final image was:

```text
nginx:1.24
```

So the rollback returned the Deployment to the previous Nginx image version.

---

# 14. Cleanup

After completing the exercises, I removed the Deployment:

```bash
kubectl delete deployment nginx-deployment -n dev
```

Then removed the standalone Pods:

```bash
kubectl delete pod nginx-dev -n dev
kubectl delete pod nginx-staging -n staging
```

Finally, I deleted the custom namespaces:

```bash
kubectl delete namespace dev staging production
```

Kubernetes confirmed:

```text
namespace "dev" deleted
namespace "staging" deleted
namespace "production" deleted
```

I verified the remaining namespaces:

```bash
kubectl get namespaces
```

The custom namespaces were gone.

I also checked:

```bash
kubectl get pods -A
```

Only the cluster/system Pods remained.

---

# 15. Important Commands

## Namespace commands

```bash
kubectl get namespaces
kubectl create namespace dev
kubectl get pods -n dev
kubectl get pods -A
kubectl delete namespace dev
```

## Deployment commands

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments -n dev
kubectl get pods -n dev
kubectl delete deployment nginx-deployment -n dev
```

## Scaling

```bash
kubectl scale deployment/nginx-deployment --replicas=4 -n dev
```

## Rolling update

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev
kubectl rollout status deployment/nginx-deployment -n dev
```

## Rollout history and rollback

```bash
kubectl rollout history deployment/nginx-deployment -n dev
kubectl rollout undo deployment/nginx-deployment -n dev
```

## Check image

```bash
kubectl describe deployment nginx-deployment -n dev | grep Image
```

---

# 16. Key Concepts to Remember

### Namespace

A logical boundary used to organize and isolate Kubernetes resources.

```text
Cluster
├── dev
├── staging
└── production
```

### Deployment

A Kubernetes controller that manages the desired state of an application and its Pods.

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

### Replica

A copy of a Pod managed by the Deployment.

```yaml
replicas: 3
```

means Kubernetes should maintain three matching Pods.

### Selector

Connects the Deployment to the Pods it manages.

```yaml
selector:
  matchLabels:
    app: nginx
```

must match:

```yaml
template:
  metadata:
    labels:
      app: nginx
```

### Self-healing

If a managed Pod disappears:

```text
3 desired
   ↓
1 Pod deleted
   ↓
2 running
   ↓
ReplicaSet creates replacement
   ↓
3 running again
```

### Scaling

Changes the desired number of replicas.

```text
1 → 4 → 10 → 1
```

### Rolling update

Gradually replaces Pods using the old configuration with Pods using the new configuration.

### Rollback

Returns a Deployment to an earlier revision.

---

# 17. Day 52 Architecture

The complete flow can be remembered like this:

```text
                    Kubernetes Cluster
                           │
             ┌─────────────┼─────────────┐
             │             │             │
          dev          staging       production
             │
             ▼
        Deployment
             │
             ▼
        ReplicaSet
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
      Pod   Pod   Pod
       │     │     │
     nginx nginx nginx
```

If one Pod is deleted:

```text
Deployment
     │
ReplicaSet
     │
     ├── Pod
     ├── Pod
     └── Pod ❌ deleted
             │
             ▼
       ReplicaSet detects
       fewer Pods than desired
             │
             ▼
        New Pod created
```

---

# 18. What I Practiced

- Listed Kubernetes namespaces
- Inspected `kube-system`
- Created `dev` and `staging` namespaces
- Created `production` using a YAML manifest
- Ran Pods in different namespaces
- Used `kubectl get pods -A`
- Created an Nginx Deployment
- Fixed a YAML indentation error
- Verified three Deployment replicas
- Deleted Deployment-managed Pods and observed self-healing
- Scaled the Deployment up and down
- Performed an Nginx image rolling update
- Checked rollout status and history
- Performed a rollback
- Verified the rolled-back Nginx image
- Cleaned up the resources

---

# 19. Quick Interview Revision

### What is a Namespace?

A Namespace logically separates and organizes Kubernetes resources inside a cluster.

### Why use a Deployment instead of creating Pods directly?

A Deployment manages the desired number of Pods and provides features such as self-healing, scaling, rolling updates, and rollbacks.

### What happens if a Deployment-managed Pod is deleted?

The ReplicaSet associated with the Deployment detects that the actual number of Pods is below the desired number and creates a replacement Pod.

### Why does the replacement Pod have a different name?

The deleted Pod is gone. Kubernetes creates a new Pod from the Deployment's Pod template, so the new Pod receives a new generated name.

### What does `replicas: 3` mean?

It means the Deployment should maintain three matching Pod replicas.

### What is the purpose of `selector.matchLabels`?

It tells the Deployment which Pods it manages. The selector must match the labels in the Pod template.

### What is a rolling update?

A process where Kubernetes gradually replaces old Pods with new Pods using the updated Deployment configuration.

### What does `kubectl rollout undo` do?

It rolls the Deployment back to a previous revision.

### What is the difference between `-n` and `-A`?

```bash
-n dev
```

targets one namespace.

```bash
-A
```

lists resources across all namespaces.

---

# Day 52 Takeaway

The biggest concept from this day is:

> **A Pod is the workload, but a Deployment manages the desired state of those Pods.**

Namespaces help organize resources:

```text
Namespace → Organization
```

Deployments manage application replicas:

```text
Deployment → ReplicaSet → Pods
```

And Kubernetes continuously works to keep the actual state aligned with the desired state.
