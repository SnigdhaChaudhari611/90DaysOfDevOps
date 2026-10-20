# Day 59: Helm, the Kubernetes Package Manager

## What is Helm?

Helm is a package manager for Kubernetes. It helps package, configure,
install, upgrade, and roll back applications without manually
maintaining every Kubernetes manifest for each deployment.

Helm does not replace Kubernetes manifests. It uses templates and values
to generate them, then manages the application's release in the cluster.

### Three core concepts

-   **Chart:** A package containing Kubernetes manifest templates,
    default values, and chart metadata.
-   **Release:** An installed instance of a chart. The same chart can be
    installed as multiple releases with different names and
    configurations.
-   **Repository:** A location where charts are published and can be
    discovered.

## 1. Install and explore Helm

Verify the installation and inspect the Helm environment:

``` bash
helm version
helm env
```

In this lab, Helm was installed on Ubuntu using Snap:

``` bash
sudo snap install helm --classic
```

The installed version was Helm `v4.3.1`.

## 2. Add a chart repository and search

Add the Bitnami chart repository and refresh its chart information:

``` bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Search for NGINX charts:

``` bash
helm search repo nginx
```

Count the charts returned by the Bitnami repository:

``` bash
helm search repo bitnami/ --output json | jq 'length'
```

The command returned `144` charts in this lab. Repository contents can
change over time.

## 3. Install NGINX from a chart

Install NGINX with the release name `my-nginx`:

``` bash
helm install my-nginx bitnami/nginx
```

Inspect the release and its resources:

``` bash
helm list
helm status my-nginx
helm get manifest my-nginx
kubectl get all
kubectl get pods
kubectl get services
```

Helm generated and installed the resources defined by the chart. The
default deployment initially had one replica.

The Service was configured as `LoadBalancer`, but its external IP
remained `<pending>` in this practice cluster because the environment
did not provision a cloud load balancer. A `LoadBalancer` Service does
not automatically receive an external IP in every Kubernetes
environment.

## 4. Customize a chart with values

A chart's values configure how its templates are rendered. Individual
values can be supplied with `--set`, or collected in a YAML file and
supplied with `-f`.

### Set values on the command line

``` bash
helm install nginx-custom bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort
```

Here, `replicaCount=3` requests three replicas, and
`service.type=NodePort` configures the Service as a NodePort Service.

### Create `custom-values.yaml`

``` yaml
replicaCount: 2

service:
  type: NodePort

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

What these settings mean:

-   `replicaCount: 2` configures two application replicas.
-   `service.type: NodePort` configures the Service type.
-   `resources.requests` specifies the CPU and memory requested for
    scheduling.
-   `resources.limits` sets the container's CPU and memory limits.

Install a separate release using the file:

``` bash
helm install nginx-file bitnami/nginx -f custom-values.yaml
```

Check the release's configured overrides and verify the resources:

``` bash
helm get values nginx-file
kubectl get deployments
kubectl get services
kubectl get pods
```

The `-f` option supplies configuration from a file. The `--set` option
is convenient for individual overrides. Nested values use dot notation,
such as `service.type=NodePort`.

## 5. Upgrade and roll back a release

Upgrade the original `my-nginx` release to five replicas:

``` bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=5
```

Inspect its history:

``` bash
helm history my-nginx
kubectl get deployments
kubectl get pods
kubectl get events --sort-by=.lastTimestamp
```

During this lab, one of the five Pods remained `Pending` because the
single-node cluster did not have enough available CPU. The event
reported `Insufficient cpu`. This was a cluster-capacity issue, not a
Helm syntax error. Helm manages the release, but Kubernetes still needs
enough resources to schedule and run the Pods.

Roll back to revision 1:

``` bash
helm rollback my-nginx 1
helm history my-nginx
kubectl get deployments
kubectl get pods
```

The rollback returned the release to the earlier configuration, and the
Deployment returned to one ready replica.

**Important:** A rollback does not overwrite or erase the newer
revision. It creates a new revision using the selected revision's
configuration. In this lab, revision 1 was the original install,
revision 2 was the upgrade, and revision 3 was the rollback.

## 6. Create and deploy a chart of your own

Create a starter chart:

``` bash
helm create my-app
```

A generated chart includes files and directories such as:

``` text
my-app/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── serviceaccount.yaml
    ├── _helpers.tpl
    └── NOTES.txt
```

-   `Chart.yaml` contains chart metadata, including its name and
    version.
-   `values.yaml` contains default configuration values.
-   `templates/` contains Kubernetes manifests with template
    expressions.
-   `charts/` is used for chart dependencies.

### Configure the chart

The relevant values in `my-app/values.yaml` were set to:

``` yaml
replicaCount: 3

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25"
```

The Deployment template uses expressions such as:

``` yaml
replicas: {{ .Values.replicaCount }}
```

and:

``` yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

Helm replaces these expressions with values when it renders the
template. For this configuration, the Deployment renders with three
replicas and the image `nginx:1.25`.

### Validate and preview before installing

``` bash
helm lint my-app
helm template my-release ./my-app
```

`helm lint` checks the chart for common issues. It passed in this lab;
the optional chart icon was noted as a recommendation.

`helm template` renders the Kubernetes manifests locally. It does
**not** install the resources into the cluster.

To inspect only the replica and image lines:

``` bash
helm template my-release ./my-app | grep -E 'replicas:|image:'
```

The output included `replicas: 3` and `image: "nginx:1.25"`. The chart's
test-connection Pod also rendered a `busybox` image, which is separate
from the NGINX application container.

### Install and upgrade your chart

Install the chart:

``` bash
helm install my-release ./my-app
kubectl get deployments
kubectl get pods
helm status my-release
```

The Deployment had three ready replicas.

Upgrade the release to five replicas:

``` bash
helm upgrade my-release ./my-app --set replicaCount=5
helm history my-release
kubectl get deployments
kubectl get pods
```

The Deployment reached five ready replicas. The release history showed
revision 1 as `superseded` and revision 2 as `deployed`.

### Key difference between common commands

  ----------------------------------------------------------------------------------
  Command                                        Purpose
  ---------------------------------------------- -----------------------------------
  `helm template my-release ./my-app`            Render manifests locally without
                                                 installing

  `helm install my-release ./my-app`             Install a chart as a release

  `helm upgrade my-release ./my-app`             Upgrade an existing release

  `helm upgrade --install my-release ./my-app`   Install if the release does not
                                                 exist, otherwise upgrade

  `helm history my-release`                      Show release revisions

  `helm rollback my-release 1`                   Roll back to revision 1's
                                                 configuration
  ----------------------------------------------------------------------------------

## 7. Clean up

Uninstall the releases created in this lab:

``` bash
helm uninstall my-nginx
helm uninstall nginx-custom
helm uninstall nginx-file
helm uninstall my-release
```

Verify that no deployed releases remain in the current namespace:

``` bash
helm list
```

To check all namespaces:

``` bash
helm list --all-namespaces
```

If release history needs to be retained for auditing, uninstall a
release with `--keep-history`:

``` bash
helm uninstall my-release --keep-history
```

This removes the release's deployed resources while retaining its
release history. A release with retained history may be visible with:

``` bash
helm list --uninstalled
```

Remove the local chart directory and custom values file only after you
no longer need them:

``` bash
rm -rf my-app
rm -f custom-values.yaml
```

These commands permanently remove the specified local files and
directory, so confirm you are in the correct working directory first.

## Key takeaways

-   A **chart** is the package; a **release** is an installed instance
    of that chart.
-   `values.yaml` and `--set` customize chart templates.
-   `helm template` renders YAML locally, while `helm install` installs
    the release.
-   `helm upgrade` updates a release, and `helm rollback` creates a new
    revision based on an earlier configuration.
-   Helm can report a release as deployed even when a Pod is `Pending`;
    Kubernetes scheduling and cluster capacity still matter.
-   Helm makes Kubernetes manifests reusable and easier to manage. It
    does not eliminate the manifests themselves.
