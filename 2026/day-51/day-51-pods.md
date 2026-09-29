# Day 51: Kubernetes Manifests and Your First Pods

## What I Learned

Kubernetes resources can be defined using YAML manifests. A Pod manifest describes the desired state of a Pod, including its metadata and containers.

### Basic Pod Manifest Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

### Four Important Fields

| Field | Purpose |
|---|---|
| `apiVersion` | API version used for the resource |
| `kind` | Type of Kubernetes resource |
| `metadata` | Name, labels, namespace and other identifying information |
| `spec` | Desired configuration of the resource |

For a Pod, `spec` contains the container configuration.

---

## Task 1: Create an Nginx Pod

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

### Create the Pod

```bash
kubectl apply -f nginx-pod.yaml
```

### Verify

```bash
kubectl get pods
kubectl get pods -o wide
```

Useful commands for inspecting the Pod:

```bash
kubectl describe pod nginx-pod
kubectl logs nginx-pod
```

### Test from Inside the Container

```bash
kubectl exec -it nginx-pod -- /bin/bash
```

Then:

```bash
curl localhost:80
```

The Nginx welcome page was returned successfully.

Exit the container:

```bash
exit
```

---

## Task 2: Create a BusyBox Pod

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  labels:
    app: busybox
    environment: dev
spec:
  containers:
    - name: busybox
      image: busybox:latest
      command: ["sh", "-c", "echo hello from busybox"]
```

Create it:

```bash
kubectl apply -f busybox-pod.yaml
```

Check its status:

```bash
kubectl get pods
```

Check logs:

```bash
kubectl logs busybox-pod
```

### Why BusyBox Exited

The command only printed a message and then finished:

```text
hello from busybox
```

Because the main container process exited, Kubernetes restarted the container and it eventually showed `CrashLoopBackOff`.

A long-running command such as:

```yaml
command: ["sh", "-c", "echo hello from busybox && sleep 3600"]
```

would keep the container running.

---

## Task 3: Imperative vs Declarative Kubernetes

### Imperative

Create a Pod directly from the command line:

```bash
kubectl run redis-pod --image=redis:latest
```

Inspect the generated resource:

```bash
kubectl get pod redis-pod -o yaml
```

Imperative approach:

```text
Tell Kubernetes what action to perform.
```

### Declarative

Define the desired state in a YAML file:

```bash
kubectl apply -f nginx-pod.yaml
```

Declarative approach:

```text
Define the desired state and let Kubernetes reconcile it.
```

This approach is easier to version-control and reuse.

### Generate YAML Without Creating the Pod

```bash
kubectl run test-pod --image=nginx --dry-run=client -o yaml
```

This generates the YAML locally without creating the resource.

---

## Task 4: Validate Kubernetes Manifests

### Client-Side Dry Run

```bash
kubectl apply -f nginx-pod.yaml --dry-run=client
```

This validates the manifest locally without sending the resource to the Kubernetes API server.

### Server-Side Dry Run

```bash
kubectl apply -f nginx-pod.yaml --dry-run=server
```

This sends the request to the API server for validation without actually persisting the resource.

### What I Practiced

I intentionally introduced manifest errors and used validation to understand the difference between:

- YAML syntax errors
- Kubernetes schema/field errors
- Missing required fields

For example, server-side validation caught a missing required container image.

---

## Task 5: Labels and Filtering

Labels are key-value pairs attached to Kubernetes resources.

Example:

```yaml
labels:
  app: nginx
  environment: dev
  team: ps7
```

### Show Labels

```bash
kubectl get pods --show-labels
```

### Filter by Label

```bash
kubectl get pods -l app=nginx
kubectl get pods -l environment=dev
kubectl get pods -l environment=production
```

### Add a Label

```bash
kubectl label pod nginx-pod environment=production
```

### Remove a Label

```bash
kubectl label pod nginx-pod environment-
```

### Pod with Multiple Labels

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: dev
    team: ps7
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

This was applied successfully and the Pod reached `Running`.

---

## Task 6: Cleanup

Delete the Pods:

```bash
kubectl delete pod nginx-pod
kubectl delete pod busybox-pod
kubectl delete pod redis-pod
```

Verify:

```bash
kubectl get pods
```

The test Pods were removed successfully.

### Important

A standalone Pod is not automatically recreated after deletion.

For production workloads, resources such as Deployments are normally used so Kubernetes can maintain the desired number of Pod replicas.

---

# Important Commands From Day 51

| Command | Purpose |
|---|---|
| `kubectl apply -f file.yaml` | Create or update a resource from a manifest |
| `kubectl get pods` | List Pods |
| `kubectl get pods -o wide` | Show additional Pod details |
| `kubectl describe pod <name>` | Detailed Pod information and events |
| `kubectl logs <pod>` | View container logs |
| `kubectl exec -it <pod> -- /bin/bash` | Open a shell inside a container |
| `kubectl get pod <name> -o yaml` | View the resource YAML |
| `kubectl run ...` | Create a resource imperatively |
| `kubectl apply --dry-run=client` | Validate locally |
| `kubectl apply --dry-run=server` | Validate through the API server |
| `kubectl get pods --show-labels` | Display Pod labels |
| `kubectl get pods -l key=value` | Filter Pods using labels |
| `kubectl label pod ...` | Add or modify a label |
| `kubectl delete pod <name>` | Delete a Pod |

---

# Key Takeaways

1. Kubernetes manifests describe the desired state of resources.
2. `apiVersion`, `kind`, `metadata`, and `spec` are the main fields used in a Pod manifest.
3. `metadata.labels` are useful for organizing and selecting resources.
4. `kubectl apply -f` is the main declarative workflow.
5. `kubectl run` is an imperative way to create a resource.
6. `--dry-run=client` validates locally, while `--dry-run=server` validates through the API server.
7. A container that exits can be restarted repeatedly and may result in `CrashLoopBackOff`.
8. `kubectl describe`, `logs`, and `exec` are useful for inspecting a Pod.
9. Standalone Pods are not the normal way to manage production workloads.
10. Deployments are used when Kubernetes needs to maintain and recreate Pod replicas.

## Day 51 Flow

```text
Write YAML
    ↓
kubectl apply -f
    ↓
API Server
    ↓
Kubernetes creates Pod
    ↓
Scheduler selects Node
    ↓
kubelet manages Pod
    ↓
Container Runtime starts container
    ↓
Pod Running
```
