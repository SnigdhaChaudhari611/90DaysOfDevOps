# Day 56: Kubernetes StatefulSets

## Objective

Understand how Kubernetes StatefulSets manage applications that need
stable identities and persistent storage. In this lab, I created a
Headless Service and a StatefulSet, verified stable pod DNS, tested
persistent data across pod deletion, scaled the StatefulSet up and down,
and cleaned up the resources.

## 1. What is a StatefulSet?

A **StatefulSet** is a Kubernetes workload controller for applications
that need a stable identity, predictable pod names, ordered
deployment/scaling, or persistent storage associated with each replica.

Unlike a Deployment, a StatefulSet gives each pod a stable ordinal name.
For a StatefulSet named `web`, the pods are named:

-   `web-0`
-   `web-1`
-   `web-2`

If `web-1` is deleted, Kubernetes can recreate it with the same name,
`web-1`. When configured with persistent volume claims, the replacement
pod can reconnect to the same claim and recover its data.

### When should I use a StatefulSet vs a Deployment?

Use a **Deployment** for stateless, interchangeable replicas such as web
frontends or APIs where any pod can handle any request.

Use a **StatefulSet** when replicas need stable identities and/or stable
storage, such as database members, distributed systems, and clustered
applications. A StatefulSet does not automatically make an application
highly available or manage database replication; the application still
needs to support those behaviors.

## 2. StatefulSet vs Deployment

  ------------------------------------------------------------------------
  Feature                 Deployment              StatefulSet
  ----------------------- ----------------------- ------------------------
  Typical workload        Stateless applications  Stateful or
                                                  identity-sensitive
                                                  applications

  Pod identity            Pods are                Stable ordinal names,
                          interchangeable and     such as `web-0` and
                          usually have generated  `web-1`
                          names                   

  Pod replacement         A replacement pod       Replacement keeps the
                          usually gets a          same ordinal name
                          different generated     
                          name                    

  Storage                 Can use volumes, but    `volumeClaimTemplates`
                          does not create a       can create a separate
                          separate PVC per        PVC for each pod
                          replica automatically   

  Network identity        Usually accessed        Can provide stable
                          through a Service       per-pod DNS when used
                                                  with a Headless Service

  Creation and scaling    Does not provide        By default, creates and
  order                   StatefulSet's ordered   removes pods in ordinal
                          identity guarantees     order

  Common examples         Web applications,       Databases and clustered
                          stateless APIs, workers systems that require
                                                  stable identity/storage
  ------------------------------------------------------------------------

## 3. How the main components work together

### Headless Service

A normal ClusterIP Service provides a virtual IP for reaching a group of
pods. A **Headless Service** sets `clusterIP: None`, so Kubernetes does
not allocate a virtual cluster IP for the Service.

When the Headless Service selects StatefulSet pods, DNS can return
individual pod addresses. This helps clients discover each replica
directly.

In this lab, the Service was named `headless-service` and selected pods
with the label `app: nginx`.

### Stable DNS

With StatefulSet `web`, namespace `default`, and Headless Service
`headless-service`, the pod DNS names follow this pattern:

``` text
<pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

Examples:

``` text
web-0.headless-service.default.svc.cluster.local
web-1.headless-service.default.svc.cluster.local
web-2.headless-service.default.svc.cluster.local
```

The DNS name remains tied to the pod's stable identity, although the
pod's IP address can change if it is recreated. DNS resolution depends
on the Service selector and the pod being discoverable through that
Service.

### `volumeClaimTemplates`

`volumeClaimTemplates` tells a StatefulSet to create a separate
PersistentVolumeClaim (PVC) for each pod. With the template named
`web-data`, the resulting claims are:

``` text
web-data-web-0
web-data-web-1
web-data-web-2
```

When the StatefulSet was scaled to five replicas, Kubernetes also
created:

``` text
web-data-web-3
web-data-web-4
```

Each claim requested `100Mi` of storage with `ReadWriteOnce` access and
used the `standard` StorageClass in this lab. A PVC requests storage;
the StorageClass and provisioner determine how the backing volume is
supplied.

## 4. Hands-on lab: Tasks 1--7

> Resource names and outputs below reflect this lab. Pod IP addresses
> can differ between runs.

### Task 1: Compare Deployment and StatefulSet pod identities

**Goal:** Understand the difference between interchangeable Deployment
replicas and StatefulSet replicas with stable identities.

A Deployment's pods typically have generated names, for example
`nginx-deployment-<replica-set-hash>-<suffix>`. StatefulSet pods use
stable ordinal names, such as `web-0`, `web-1`, and `web-2`.

**Key takeaway:** StatefulSet pod names are predictable and remain the
same when the corresponding pod is recreated.

### Task 2: Create a Headless Service

The Service used `clusterIP: None` and selected pods with `app: nginx`.

Example manifest:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Apply and verify it:

``` bash
kubectl apply -f headless-service.yaml
kubectl get svc headless-service
```

**Important correction from the lab:** Kubernetes expects the value
`None` with a capital `N`; lowercase `none` was rejected.

Before the StatefulSet pods existed, the Service had no pod endpoints.
Once matching pods were ready, DNS could discover the individual pods.

### Task 3: Create the StatefulSet and per-pod PVCs

The StatefulSet was named `web`, used `serviceName: headless-service`,
and had three replicas. The `volumeClaimTemplates` section had to be
nested under `spec`.

The StorageClass in the cluster was named `standard`, not `default`. The
incorrect StorageClass caused PVCs to remain Pending until the manifest
was corrected.

Representative manifest structure:

``` yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: headless-service
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
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: web-data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: web-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: standard
        resources:
          requests:
            storage: 100Mi
```

Apply and verify:

``` bash
kubectl apply -f nginx-statefulset.yaml
kubectl get pods -l app=nginx
kubectl get pvc
```

**Observed result:** `web-0`, `web-1`, and `web-2` reached `Running`,
and their PVCs became `Bound`.

  Pod       PVC
  --------- ------------------
  `web-0`   `web-data-web-0`
  `web-1`   `web-data-web-1`
  `web-2`   `web-data-web-2`

### Task 4: Verify stable DNS resolution

A temporary BusyBox pod was used for DNS lookup:

``` bash
kubectl run busybox --image=busybox:1.36 --restart=Never --command -- sleep 3600
```

Run DNS lookups from inside it:

``` bash
kubectl exec busybox -- nslookup web-0.headless-service.default.svc.cluster.local
kubectl exec busybox -- nslookup web-1.headless-service.default.svc.cluster.local
kubectl exec busybox -- nslookup web-2.headless-service.default.svc.cluster.local
```

The observed pod IPs during this run were:

  DNS name                                             Resolved IP
  ---------------------------------------------------- ---------------
  `web-0.headless-service.default.svc.cluster.local`   `10.244.0.10`
  `web-1.headless-service.default.svc.cluster.local`   `10.244.0.12`
  `web-2.headless-service.default.svc.cluster.local`   `10.244.0.14`

These IPs are specific to this run and may be different in another
cluster or after pod recreation.

**Key takeaway:** The DNS names are stable; the IP addresses are not
guaranteed to stay the same.

If the temporary BusyBox pod is no longer needed, remove it:

``` bash
kubectl delete pod busybox
```

### Task 5: Verify that pod data survives pod deletion

Write different content to each pod's mounted directory:

``` bash
kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data from web-2' > /usr/share/nginx/html/index.html"
```

Verify the contents:

``` bash
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
kubectl exec web-2 -- cat /usr/share/nginx/html/index.html
```

Then delete `web-0`:

``` bash
kubectl delete pod web-0
```

Wait for the StatefulSet to recreate it, then check its data:

``` bash
kubectl get pods -l app=nginx
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
```

**Observed result:** The recreated `web-0` returned:

``` text
Data from web-0
```

**Answer:** The data survived because the replacement pod reused the PVC
associated with ordinal `web-0`. Deleting a pod is not the same as
deleting its PVC.

### Task 6: Scale up and scale down

Scale the StatefulSet from three replicas to five:

``` bash
kubectl scale statefulset web --replicas=5
kubectl get pods -l app=nginx
kubectl get pvc
```

**Observed result:** Kubernetes created `web-3` and `web-4`. Five PVCs
existed and were Bound, one for each ordinal from `web-0` through
`web-4`.

  Pod       PVC
  --------- ------------------
  `web-0`   `web-data-web-0`
  `web-1`   `web-data-web-1`
  `web-2`   `web-data-web-2`
  `web-3`   `web-data-web-3`
  `web-4`   `web-data-web-4`

Scale back down to three:

``` bash
kubectl scale statefulset web --replicas=3
kubectl get pods -l app=nginx
kubectl get pvc
```

**Observed result:** Only `web-0`, `web-1`, and `web-2` remained as
pods. The PVCs for `web-3` and `web-4` remained.

**Answer:** Scaling down removed the higher-ordinal pods, but their PVCs
were retained. This protects data if those replicas are later needed
again. The final state confirmed that the first three pods were running
and all five PVCs still existed.

### Task 7: Clean up resources

Delete the StatefulSet and Headless Service:

``` bash
kubectl delete statefulset web
kubectl delete service headless-service
```

Check the PVCs:

``` bash
kubectl get pvc
```

**Observed result:** The PVCs remained after the StatefulSet and Service
were deleted.

Delete the claims manually when their data is no longer needed:

``` bash
kubectl delete pvc \
  web-data-web-0 \
  web-data-web-1 \
  web-data-web-2 \
  web-data-web-3 \
  web-data-web-4
```

Verify cleanup:

``` bash
kubectl get pvc
kubectl get pods -l app=nginx
kubectl get svc
```

**Answer to the verification question:** No, the PVCs were not
automatically deleted with the StatefulSet in this lab. By default,
StatefulSet PVCs are retained when the StatefulSet is deleted or scaled
down. Kubernetes supports configurable PVC retention policies, so
behavior can differ if a policy has been explicitly set. Deleting PVCs
may permanently delete their data, depending on the volume's reclaim
policy.

## 5. Screenshots to include

Add your actual terminal screenshots in this section before publishing
the README. Capture readable terminal output, not credentials or other
sensitive information.

### Screenshot 1: StatefulSet pods

Show the three running pods and their stable names.

Suggested command:

``` bash
kubectl get pods -l app=nginx -o wide
```

![StatefulSet pods](screenshots/day-56-statefulset-pods.png)

### Screenshot 2: PVCs

Show the per-pod PVCs and their status. The most useful evidence is from
Task 6, when five claims existed and were Bound.

Suggested command:

``` bash
kubectl get pvc
```

![StatefulSet PVCs](screenshots/day-56-statefulset-pvcs.png)

### Screenshot 3: DNS resolution

Show BusyBox resolving the stable DNS names for the StatefulSet pods.

Suggested command:

``` bash
kubectl exec busybox -- nslookup web-0.headless-service.default.svc.cluster.local
kubectl exec busybox -- nslookup web-1.headless-service.default.svc.cluster.local
kubectl exec busybox -- nslookup web-2.headless-service.default.svc.cluster.local
```

![StatefulSet DNS resolution](screenshots/day-56-statefulset-dns.png)

> The image paths above are placeholders. Save your actual screenshots
> with those filenames in a `screenshots/` folder next to this Markdown
> file, or change the paths to match your repository.

## 6. Key takeaways

-   StatefulSets provide stable pod identities such as `web-0` and
    `web-1`.
-   A Headless Service (`clusterIP: None`) supports DNS discovery of
    individual StatefulSet pods.
-   A pod's DNS name can remain stable even if its IP address changes.
-   `volumeClaimTemplates` creates a separate PVC for each StatefulSet
    replica.
-   Recreating a pod can preserve its data when it reconnects to the
    same PVC.
-   Scaling down or deleting a StatefulSet does not normally remove its
    PVCs automatically.
-   PVCs should be deleted deliberately because deleting them can put
    persistent data at risk.

## 7. Cleanup checklist

-   [ ] StatefulSet `web` deleted
-   [ ] Headless Service `headless-service` deleted
-   [ ] PVCs checked after deleting the StatefulSet
-   [ ] Five exercise PVCs deleted manually
-   [ ] Remaining exercise pods checked
-   [ ] Actual screenshots added and image paths verified
