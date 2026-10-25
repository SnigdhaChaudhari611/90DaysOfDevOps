# Day 60: Kubernetes Capstone

## Objective

Deploy WordPress and MySQL on Kubernetes and practise application configuration, persistent storage, health checks, self-healing, autoscaling, and Helm.

## 1. Architecture

- **WordPress Deployment:** Runs two application replicas.
- **MySQL StatefulSet:** Runs the database with a PersistentVolumeClaim (PVC).
- **Headless Service:** Gives the MySQL StatefulSet a stable network identity.
- **NodePort Service:** Exposes WordPress on port `30080`.
- **Secret:** Supplies database credentials to MySQL and WordPress.
- **ConfigMap:** Stores WordPress database connection settings.
- **Resource requests and limits:** Define CPU and memory requirements and limits.
- **Liveness and readiness probes:** Check application health and readiness.
- **HPA:** Targets 50% average CPU utilization and allows 2–10 WordPress replicas.

The manifests for the manual deployment are in the `manifests/` directory.

## 2. Deploy and Inspect

Apply the manifests in dependency order, using the filenames in `manifests/`:

```bash
kubectl apply -f manifests/mysql-secret.yaml
kubectl apply -f manifests/mysql-service.yaml
kubectl apply -f manifests/mysql-statefulset.yaml
kubectl apply -f manifests/wordpress-configmap.yaml
kubectl apply -f manifests/wordpress-deployment.yaml
kubectl apply -f manifests/wordpress-service.yaml
kubectl apply -f manifests/wordpress-hpa.yaml
```

Check the resources:

```bash
kubectl get all -n capstone
kubectl get pvc,configmap,secret,hpa -n capstone
kubectl get pods -n capstone -o wide
```

`kubectl get all` does not show every resource type, so inspect PVCs, ConfigMaps, Secrets, and the HPA separately.

## 3. Storage and Self-Healing

MySQL uses a StatefulSet and PVC to keep database files outside the container's writable layer. A StatefulSet recreates the database Pod with the same identity and reattaches its claim when storage is available.

Inspect the claim and database Pod:

```bash
kubectl get pvc -n capstone
kubectl describe pvc -n capstone
kubectl get statefulset,pods -n capstone
```

To test WordPress Deployment self-healing, delete a WordPress Pod and watch the replacement:

```bash
kubectl delete pod -n capstone -l app=wordpress
kubectl get pods -n capstone -w
```

The Deployment controller should bring the number of ready WordPress replicas back to the desired count.

**Persistence verification:** After creating a test post in WordPress, delete the MySQL Pod, wait for `mysql-0` to return to `Running`, and check that the post remains. Record the result from your own test here; do not assume persistence is verified solely because the Pod restarted.

## 4. Resource Limits and Probes

The WordPress and MySQL manifests define CPU and memory requests and limits. Requests help Kubernetes schedule Pods; limits constrain resource consumption.

The WordPress manifest includes liveness and readiness probes using `/wp-login.php`. Readiness affects whether the Pod receives Service traffic; liveness can trigger a container restart if the health check repeatedly fails.

Inspect events and probe status when debugging:

```bash
kubectl describe pod -n capstone
kubectl get events -n capstone --sort-by=.lastTimestamp
```

## 5. Horizontal Pod Autoscaler

The HPA is configured for the WordPress Deployment with a minimum of 2 replicas, maximum of 10, and a CPU utilization target of 50%.

```bash
kubectl get hpa -n capstone
kubectl describe hpa wordpress-hpa -n capstone
kubectl top pods -n capstone
```

During verification, the HPA reported approximately `7%/50%` CPU utilization. Metrics were available and the HPA could evaluate the metric. This reading alone does not demonstrate scale-out; that requires sufficient load and observing the replica count change.

## 6. Bonus: Helm Deployment

A separate WordPress release was installed using the Bitnami WordPress chart in the `capstone-helm` namespace. The chart deployed WordPress with MariaDB.

```bash
helm list -n capstone-helm
kubectl get all -n capstone-helm
```

Both WordPress and MariaDB Pods reached `Running`. The WordPress Service was configured as `LoadBalancer`, but its external IP remained `<pending>` because the Kind cluster did not have a load-balancer implementation configured.

### Manual manifests vs Helm

- **Manual manifests:** Each Kubernetes resource is defined and maintained in an individual YAML file, giving direct control over its configuration.
- **Helm:** A chart packages related resources and supports repeatable installation, configuration through values, upgrades, and release management.

## 7. Cleanup

After saving the results, remove the manual deployment and reset the current namespace:

```bash
kubectl delete namespace capstone
kubectl config set-context --current --namespace=default
```

The Helm deployment is in a separate namespace, `capstone-helm`, and is not removed by deleting `capstone`. Clean it up separately if it is no longer needed.

## Key Learnings

- Deployments manage interchangeable application replicas; StatefulSets are useful when workloads need stable identity and persistent storage.
- Services provide stable network access to Pods.
- ConfigMaps and Secrets separate configuration from workload definitions. Kubernetes Secrets are not automatically encrypted just because their values are stored as Secrets; production clusters need appropriate RBAC and encryption-at-rest configuration.
- Probes help Kubernetes manage application health and readiness.
- HPA needs working metrics and resource requests to calculate CPU utilization.
- Helm provides a reusable way to install and manage an application composed of multiple Kubernetes resources.

## Manifests

- `manifests/mysql-secret.yaml`
- `manifests/mysql-service.yaml`
- `manifests/mysql-statefulset.yaml`
- `manifests/wordpress-configmap.yaml`
- `manifests/wordpress-deployment.yaml`
- `manifests/wordpress-service.yaml`
- `manifests/wordpress-hpa.yaml`

**Credential safety:** Keep real passwords out of Git. Use placeholder values in the example Secret manifest and provide credentials securely when deploying.
