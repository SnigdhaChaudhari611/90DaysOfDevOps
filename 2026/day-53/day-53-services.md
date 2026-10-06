# Day 53 – Kubernetes Services

## Objective

Understand how Kubernetes Services provide a stable network endpoint for Pods and how different Service types are used for internal and external access.

### Services covered

* ClusterIP
* NodePort
* LoadBalancer
* Kubernetes DNS-based Service discovery
* Service Endpoints

---

# Task 1: Deploy the Application

Created a Deployment with 3 Nginx replicas.

### `app-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

Applied the Deployment:

```bash
kubectl apply -f app-deployment.yaml
```

Verified the Pods:

```bash
kubectl get pods -o wide
```

All 3 Pods were running.

Example Pod IPs:

```text
10.244.0.3
10.244.0.7
10.244.0.6
```

The important point is that these are **Pod IPs**, not stable application endpoints. If Pods are recreated, their IPs can change.

---

# Task 2: ClusterIP Service

A ClusterIP Service provides a stable internal endpoint for Pods.

### `clusterip-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
```

Applied the Service:

```bash
kubectl apply -f clusterip-service.yaml
```

Verified:

```bash
kubectl get services -o wide
```

The Service received:

```text
web-app-clusterip   ClusterIP   10.96.248.31   <none>   80/TCP
```

The Service selector is:

```text
app=web-app
```

This matches the labels on the Deployment Pods.

### Test from inside the cluster

Started a temporary BusyBox Pod:

```bash
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh
```

Inside the Pod:

```bash
wget -qO- http://web-app-clusterip
```

The Nginx welcome page was returned successfully.

This confirmed:

```text
BusyBox Pod
     |
     v
ClusterIP Service
     |
     v
Nginx Pods
```

The ClusterIP is stable even though the individual Pod IPs can change.

---

# Task 3: Kubernetes DNS

Kubernetes automatically creates DNS records for Services.

The general format is:

```text
<service-name>.<namespace>.svc.cluster.local
```

For this Service:

```text
web-app-clusterip.default.svc.cluster.local
```

Tested the short DNS name:

```bash
wget -qO- http://web-app-clusterip
```

Tested the full DNS name:

```bash
wget -qO- http://web-app-clusterip.default.svc.cluster.local
```

Both returned the Nginx welcome page.

### DNS lookup

Inside the temporary Pod:

```bash
nslookup web-app-clusterip
```

The lookup resolved to:

```text
Name:    web-app-clusterip.default.svc.cluster.local
Address: 10.96.248.31
```

The returned address matched the ClusterIP:

```text
10.96.248.31
```

This demonstrates Kubernetes Service discovery through DNS.

---

# Task 4: NodePort Service

A NodePort Service exposes an application through a port on the Kubernetes node.

### `nodeport-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Applied:

```bash
kubectl apply -f nodeport-service.yaml
```

Verified:

```bash
kubectl get services -o wide
```

Output included:

```text
web-app-nodeport   NodePort   10.96.224.188   <none>   80:30080/TCP
```

This means:

```text
Service port: 80
NodePort:     30080
Target port:  80
```

### Important distinction

The following is the Service's ClusterIP:

```text
10.96.224.188
```

The external NodePort is:

```text
30080
```

Therefore, testing NodePort access requires:

```text
<NodeIP>:30080
```

rather than:

```text
<ClusterIP>:80
```

### Find the Node IP

```bash
kubectl get nodes -o wide
```

The node had:

```text
INTERNAL-IP: 172.18.0.2
```

Tested NodePort access:

```bash
curl http://172.18.0.2:30080
```

The Nginx welcome page was returned successfully.

This confirmed:

```text
Client
  |
  | 172.18.0.2:30080
  v
NodePort Service
  |
  v
Nginx Pods
```

### Verify Endpoints

```bash
kubectl get endpoints web-app-nodeport
```

The Service had these endpoints:

```text
10.244.0.3:80
10.244.0.6:80
10.244.0.7:80
```

This confirmed that the Service selector correctly matched all 3 application Pods.

> Note: Kubernetes now recommends EndpointSlice APIs over the older Endpoints API in newer Kubernetes versions.

---

# Task 5: LoadBalancer Service

A LoadBalancer Service is normally used to expose an application through an external load balancer provided by a cloud environment.

### `loadbalancer-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
```

Applied:

```bash
kubectl apply -f loadbalancer-service.yaml
```

Verified:

```bash
kubectl get services -o wide
```

Output:

```text
web-app-loadbalancer   LoadBalancer   10.96.58.217   <pending>   80:32493/TCP
```

### Why is EXTERNAL-IP `<pending>`?

The Kubernetes cluster used for this exercise is a local cluster and does not have a cloud provider automatically provisioning an external load balancer.

Therefore:

```text
EXTERNAL-IP = <pending>
```

is expected.

The Service was still created successfully.

---

# Task 6: Verify LoadBalancer Configuration

Used:

```bash
kubectl describe service web-app-loadbalancer
```

Important values:

```text
Type:        LoadBalancer
IP:          10.96.58.217
TargetPort:  80/TCP
NodePort:    32493/TCP
Endpoints:   10.244.0.6:80
             10.244.0.3:80
             10.244.0.7:80
```

This demonstrated that the LoadBalancer Service also received:

* A ClusterIP
* A NodePort
* Pod endpoints

Conceptually:

```text
External Load Balancer
          |
          v
      NodePort
          |
          v
      ClusterIP
          |
          v
     Nginx Pods
```

In a cloud-managed Kubernetes environment, the LoadBalancer Service can request an external load balancer from the cloud provider.

---

# Service Types Comparison

| Service Type | Access                        | Main Use Case                                          |
| ------------ | ----------------------------- | ------------------------------------------------------ |
| ClusterIP    | Inside the cluster            | Internal service-to-service communication              |
| NodePort     | Through `<NodeIP>:<NodePort>` | Development, testing and direct node access            |
| LoadBalancer | External load balancer        | Exposing applications externally in cloud environments |

### ClusterIP

```text
Pod → ClusterIP Service → Application Pods
```

Default Service type.

Used when the application only needs to be accessed from inside the cluster.

### NodePort

```text
Client → NodeIP:NodePort → Service → Pods
```

Exposes the Service on a port on each node.

In this task:

```text
NodePort = 30080
```

### LoadBalancer

```text
External Client
      |
      v
Cloud Load Balancer
      |
      v
Service
      |
      v
Pods
```

Used when an external load balancer is required, particularly in cloud environments.

---

# Service Selectors

The Services used:

```yaml
selector:
  app: web-app
```

The Pods had:

```yaml
labels:
  app: web-app
```

The selector must match the Pod labels.

If the selector does not match any Pods, the Service will have no usable endpoints.

---

# Endpoints

Endpoints represent the Pod IP addresses and ports currently receiving traffic through a Service.

Checked using:

```bash
kubectl get endpoints web-app-nodeport
```

Result:

```text
10.244.0.3:80
10.244.0.6:80
10.244.0.7:80
```

These corresponded to the 3 Nginx Pods.

This is useful when troubleshooting a Service:

```text
Service exists
       |
       v
Does it have endpoints?
       |
       +-- No → Check selector and Pod labels
       |
       +-- Yes → Check networking/application
```

---

# Common Mistakes During the Task

### 1. Incorrect `kubectl` output option

Used:

```bash
kubectl get services -o wode
```

Correct:

```bash
kubectl get services -o wide
```

### 2. Testing the NodePort using the ClusterIP

The NodePort Service had:

```text
ClusterIP: 10.96.224.188
NodePort:  30080
```

Testing:

```bash
curl 10.96.224.188
```

failed because this was testing the ClusterIP from outside the appropriate context.

The correct NodePort test was:

```bash
curl http://172.18.0.2:30080
```

which successfully returned the Nginx page.

### 3. Typo in LoadBalancer YAML

Initially used:

```yaml
metadat:
```

instead of:

```yaml
metadata:
```

This caused:

```text
resource name may not be empty
```

After correcting it to `metadata`, the Service was created successfully.

---

# Cleanup

Removed the Deployment and all three Services:

```bash
kubectl delete -f app-deployment.yaml
kubectl delete -f clusterip-service.yaml
kubectl delete -f nodeport-service.yaml
kubectl delete -f loadbalancer-service.yaml
```

Verified:

```bash
kubectl get pods
```

Result:

```text
No resources found in default namespace.
```

Verified Services:

```bash
kubectl get services
```

Only the default Kubernetes Service remained:

```text
kubernetes   ClusterIP   10.96.0.1   <none>   443/TCP
```

---

# Key Takeaways

1. Pods have temporary IP addresses, so applications should not depend directly on Pod IPs.
2. A Service provides a stable network endpoint for a group of Pods.
3. Service selectors connect Services to Pods through matching labels.
4. ClusterIP is used for internal cluster communication.
5. NodePort exposes a Service through a port on the Kubernetes nodes.
6. LoadBalancer is used to provision external load balancing when supported by the environment.
7. Kubernetes automatically provides DNS names for Services.
8. Endpoints show which Pods are currently receiving Service traffic.
9. A LoadBalancer Service normally also has a ClusterIP and NodePort unless configured otherwise.
10. `<pending>` for the LoadBalancer external IP is expected when no load-balancer integration is available.
