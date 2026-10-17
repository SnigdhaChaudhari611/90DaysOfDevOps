# Day 57: Kubernetes Resource Requests, Limits, and Probes

Kubernetes uses resource requests, resource limits, and health probes to manage workloads. In this lab, I tested how Pods behave when resources are unavailable, a container exceeds its memory limit, and health checks fail.

## 1. Resource requests vs limits

| Setting | Purpose | What Kubernetes does |
|---|---|---|
| **Request** | The amount of CPU or memory a container is expected to need | The scheduler uses requests when deciding whether a node has enough allocatable resources for the Pod. |
| **Limit** | The maximum CPU or memory a container is allowed to use | The container runtime and kernel enforce the limit while the container runs. |

Requests are used for **scheduling**; limits are used for **runtime enforcement**.

### What happens when a limit is exceeded?

- **CPU limit:** CPU usage is throttled when the container tries to use more CPU time than its limit allows. The container is not normally terminated just for exceeding its CPU limit.
- **Memory limit:** If the container uses more memory than its limit and the system cannot reclaim enough memory, it may be terminated by the out-of-memory (OOM) mechanism. Kubernetes can report the termination reason as `OOMKilled`.
- **Request cannot be satisfied:** If a Pod requests more CPU or memory than any available node can provide, it remains `Pending`. Use `kubectl describe pod` to inspect scheduling events.

### Resource requests and limits in my lab

I created an Nginx Pod with CPU and memory requests and limits, then inspected its configuration with `kubectl describe pod`.

![Resource requests and limits](screenshots/resource-requests-limits.png)

## 2. OOMKilled: memory limit exceeded

I configured a stress container with a memory limit of `100Mi` and asked it to allocate `200M`. The container was terminated, and Kubernetes reported `OOMKilled` with exit code `137`.

![OOMKilled Pod](screenshots/oomkilled-pod.png)

![OOMKilled reason and exit code](screenshots/oomkilled-reason-exit-code.png)

**Key point:** `OOMKilled` is the container's termination reason. Exit code `137` commonly indicates the process received `SIGKILL`, but the exit code alone does not prove an OOM event. Check the container's termination reason and Pod events.

## 3. Pending: request exceeds available node resources

I created a Pod requesting `100` CPU cores and `128Gi` of memory. The node could not satisfy those requests, so the Pod remained `Pending`.

The scheduling events showed `FailedScheduling` with `Insufficient cpu` and `Insufficient memory`.

![Pending Pod](screenshots/pending-pod.png)

![Pending Pod scheduling events](screenshots/pending-scheduling-events.png)

**Key point:** a Pod can be accepted by the API server but remain unscheduled because no node has enough available resources to satisfy its requests.

## 4. Liveness, readiness, and startup probes

| Probe | Question it answers | If it fails |
|---|---|---|
| **Liveness** | Is the container still healthy enough to keep running? | After the configured consecutive failures, Kubernetes restarts the container. |
| **Readiness** | Is the application ready to receive traffic? | The Pod is marked unready and is excluded from the ready endpoints of matching Services. The container is not restarted solely because readiness fails. |
| **Startup** | Has the application finished starting? | Kubernetes continues checking until success or the configured failure threshold is reached. If it fails too many times, the container is terminated and may be restarted. While the startup probe has not succeeded, liveness and readiness probes are held back. |

### Liveness probe: restart an unhealthy container

I configured a liveness probe to check for `/tmp/healthy`. The container removed that file after 30 seconds. The probe then failed repeatedly, and Kubernetes restarted the container. Continued failures led to `CrashLoopBackOff`.

`CrashLoopBackOff` is a restart-backoff state, not the root cause. Inspect the container state, logs, and Pod events to determine why it keeps failing.

![Liveness probe events](screenshots/liveness-probe-events.png)

### Readiness probe: running does not mean ready

I exposed an Nginx Pod through `readiness-svc`, then removed `/usr/share/nginx/html/index.html`, causing its HTTP readiness probe to fail. The Pod stopped being ready and disappeared from the Service's endpoints, while its restart count remained `0`.

![Readiness probe and Service endpoints](screenshots/readiness-probe-endpoints.png)

**Key point:** readiness controls whether a Pod receives Service traffic; it does not restart the container.

### Startup probe: allow enough time to start

I configured a container that needed about 20 seconds to create `/tmp/started`. With `failureThreshold: 12` and `periodSeconds: 5`, it had enough time to start successfully. Reducing the threshold to `2` gave it a much shorter startup window and caused repeated restarts before it could finish starting.

![Startup probe results](screenshots/startup-probe-results.png)

**Key point:** set startup probe thresholds according to the application's realistic startup time. Probe timing is approximate because checks run on a schedule and can take time to complete.

## 5. Commands used to investigate

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get endpoints <service-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

On Kubernetes v1.33 and later, the legacy `Endpoints` API is deprecated; `EndpointSlice` is preferred. To inspect EndpointSlices for a Service:

```bash
kubectl get endpointslice
```

## Key takeaways

- **Requests** influence scheduling; **limits** constrain resource use at runtime.
- CPU limit overruns generally cause throttling; memory limit overruns can result in `OOMKilled`.
- A Pod may stay `Pending` when its resource requests cannot be scheduled.
- Liveness probes can trigger container restarts, readiness probes control traffic eligibility, and startup probes protect slow-starting applications.
- `Running` does not necessarily mean `Ready`, and a status alone rarely explains the root cause. Inspect events and container details.

**Day 57/90**
