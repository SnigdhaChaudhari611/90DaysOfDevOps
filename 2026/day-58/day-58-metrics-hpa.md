# Day 58: Metrics Server and Horizontal Pod Autoscaling (HPA)

## Objective

- Install Metrics Server and verify resource metrics with `kubectl top`.
- Inspect CPU and memory usage across nodes and pods.
- Deploy a CPU-based workload and configure a Horizontal Pod Autoscaler (HPA).
- Generate load and observe automatic scaling.
- Configure HPA behavior with `autoscaling/v2`.
- Clean up the test workload while keeping Metrics Server installed.

## 1. Metrics Server

Metrics Server collects resource usage data from Kubernetes nodes and pods and makes it available through the Kubernetes Metrics API. Commands such as `kubectl top nodes` and `kubectl top pods` use this API.

### Install and verify

Applied the official Metrics Server manifest:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

In this local Kind cluster, Metrics Server initially could not validate the kubelet certificate because the certificate did not contain the node IP as an IP SAN. For this lab only, I added the following argument to the Metrics Server Deployment:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

> **Security note:** `--kubelet-insecure-tls` disables kubelet certificate verification. It was used only to work around the certificate issue in this local Kind lab. Do not use this as a production fix; configure valid kubelet certificates instead.

Verified that Metrics Server was running and that the Metrics API worked:

```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
```

Observed after cleanup:

```text
NAME                                      CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
devops-practice-cluster-control-plane     182m         9%       1279Mi          16%
```

## 2. Inspect node and pod metrics

Commands used:

```bash
kubectl top nodes
kubectl top pods -A
kubectl top pods -A --sort-by=cpu
```

In one sorted sample, the Kubernetes API server was the highest CPU consumer at `41m`. Metrics values fluctuate, so the highest consumer can differ between samples.

These commands help inspect current resource usage, but the readings are snapshots rather than long-term monitoring history.

## 3. Deploy the PHP-Apache workload

Created `php-apache-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
```

Applied the Deployment, waited for rollout, and exposed it internally:

```bash
kubectl apply -f php-apache-deployment.yaml
kubectl rollout status deployment/php-apache
kubectl expose deployment php-apache --port=80
```

Verified the workload:

```bash
kubectl get deployment php-apache
kubectl get pods -l run=php-apache
kubectl get service php-apache
kubectl top pods -l run=php-apache
```

Before generating load, the pod used approximately `1m` CPU and `9Mi` memory.

## 4. Create an HPA imperatively

Created an HPA targeting 50% CPU utilization, with between 1 and 10 replicas:

```bash
kubectl autoscale deployment php-apache \
  --cpu=50% \
  --min=1 \
  --max=10
```

The lab command using `--cpu-percent=50` also worked, but Kubernetes displayed a deprecation warning recommending `--cpu=50%`.

Checked the HPA:

```bash
kubectl get hpa
kubectl describe hpa php-apache
```

The HPA reported a CPU target of `50%` and successfully retrieved metrics (`ScalingActive: True`, reason `ValidMetricFound`).

### How CPU utilization is calculated

The workload requests `200m` CPU per pod. A 50% utilization target corresponds to an average of approximately `100m` CPU per pod.

The HPA compares observed average CPU utilization against the target and adjusts the desired replica count. A simplified replica calculation is:

```text
desiredReplicas = ceil(currentReplicas × currentMetricValue / targetMetricValue)
```

For example, if 1 replica is using 200% of the target, the basic calculation recommends:

```text
ceil(1 × 200 / 50) = 4 replicas
```

The actual result can also be affected by HPA limits, scaling policies, stabilization windows, missing metrics, and scheduling capacity.

## 5. Generate load and observe scaling

Started a BusyBox pod that repeatedly sent requests to the Service:

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

Watched the HPA:

```bash
kubectl get hpa php-apache --watch
```

Observed CPU utilization and desired replicas change during the test:

```text
cpu: 0%/50%   replicas: 1
cpu: 266%/50% replicas: 1
cpu: 392%/50% replicas: 4
cpu: 113%/50% replicas: 8
```

The Deployment subsequently reported 9 desired replicas. The highest confirmed desired replica count in the captured output was **9**; the HPA maximum was 10.

Inspected the pods and their resource usage:

```bash
kubectl get deployment php-apache
kubectl get pods -l run=php-apache
kubectl top pods -l run=php-apache
```

In the captured metrics sample, four running pods used between `191m` and `334m` CPU, and around `11–12Mi` memory each.

### Scheduling capacity matters

Some replicas remained `Pending`. The pod's Events showed:

```text
Warning  FailedScheduling
0/1 nodes are available: 1 Insufficient cpu.
```

This Kind cluster had one node. HPA increased the desired replica count in response to CPU load, but the node did not have enough allocatable CPU to schedule every pod requesting `200m`.

**Key lesson:** HPA changes the desired number of replicas; it does not add node capacity. A pod can remain `Pending` when the scheduler cannot satisfy its resource requests.

Stopped the load generator after observing scaling:

```bash
kubectl delete pod load-generator
```

After the load stopped, CPU utilization fell to `0%`, but the HPA still showed 9 desired replicas. Scale-down is not necessarily immediate because HPA uses stabilization and scaling policies to avoid reacting too quickly to short-lived changes.

## 6. Configure HPA with `autoscaling/v2`

Deleted the imperative HPA and created `php-apache-hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 15
```

Applied and inspected the HPA:

```bash
kubectl apply -f php-apache-hpa.yaml
kubectl get hpa php-apache
kubectl describe hpa php-apache
```

The description confirmed:

- CPU target utilization: `50%`.
- Minimum replicas: `1`; maximum replicas: `10`.
- Scale-up stabilization window: `0` seconds.
- Scale-up policies: up to 100% or 4 pods per 15-second period, with `selectPolicy: Max`.
- Scale-down stabilization window: `300` seconds (5 minutes).
- `ScalingActive: True` and `ValidMetricFound`.

### `autoscaling/v1` vs `autoscaling/v2`

- **`autoscaling/v1`** supports the basic HPA API and CPU utilization through the older target field.
- **`autoscaling/v2`** supports configurable metric targets and scaling behavior, including stabilization windows and scaling policies. It can also support multiple metric types.

The `autoscaling/v2` API is useful when scaling behavior needs to be explicitly controlled.

## 7. Cleanup

Deleted the test HPA, Service, Deployment, and load-generator pod:

```bash
kubectl delete hpa php-apache --ignore-not-found
kubectl delete service php-apache --ignore-not-found
kubectl delete deployment php-apache --ignore-not-found
kubectl delete pod load-generator --ignore-not-found
```

Verified the cleanup and kept Metrics Server installed:

```bash
kubectl get hpa
kubectl get deployment php-apache
kubectl get service php-apache
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
```

The HPA, Deployment, and Service were absent, Metrics Server remained `Running`, and `kubectl top nodes` continued to return metrics.

## Key takeaways

- Metrics Server supplies resource metrics used by `kubectl top` and CPU-based HPA.
- CPU utilization targets are calculated relative to CPU requests, not CPU limits.
- HPA adjusts desired replicas based on observed metrics and the target.
- `autoscaling/v2` supports configurable scale-up and scale-down behavior.
- Scaling replicas does not guarantee that every pod can be scheduled; node capacity and resource requests still matter.
- Metrics Server is a resource-metrics component, not a replacement for a full monitoring and historical-metrics system.

## Screenshots

Add terminal screenshots to this section if required by the challenge:

- `kubectl top nodes` and `kubectl top pods -A --sort-by=cpu`
- `kubectl get hpa` / `kubectl describe hpa php-apache` during scaling
- Pending pod Events showing `Insufficient cpu`
- `kubectl describe hpa php-apache` confirming the `autoscaling/v2` behavior settings
- Final cleanup showing Metrics Server running and `kubectl top nodes` working
