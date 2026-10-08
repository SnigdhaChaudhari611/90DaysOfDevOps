# Day 54: Kubernetes ConfigMaps and Secrets

## Overview

Applications need configuration such as:

* Environment names
* Feature flags
* Port numbers
* Database usernames and passwords
* API keys
* Configuration files

Hardcoding configuration inside a container image creates a problem.

If a configuration value changes, the image would need to be rebuilt.

Kubernetes provides two resources for handling application configuration:

* **ConfigMap** for non-sensitive configuration
* **Secret** for sensitive configuration

The configuration can then be provided to a Pod in two main ways:

```text
ConfigMap / Secret
        │
        ├── Environment variables
        │
        └── Volume-mounted files
```

---

# ConfigMaps

A ConfigMap stores **non-sensitive configuration data** as key-value pairs.

Example:

```text
APP_ENV=production
APP_DEBUG=false
APP_PORT=8080
```

The important idea is:

> Configuration is kept outside the container image.

This means the same container image can be used in different environments with different configuration.

---

# Task 1: Create a ConfigMap from Literals

Create a ConfigMap called `app-config`:

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080
```

Expected:

```text
configmap/app-config created
```

## What does `--from-literal` mean?

`--from-literal` means that the key-value data is provided directly in the command.

For example:

```bash
--from-literal=APP_ENV=production
```

means:

```text
Key:   APP_ENV
Value: production
```

The resulting ConfigMap contains:

```text
app-config
├── APP_ENV    → production
├── APP_DEBUG  → false
└── APP_PORT   → 8080
```

## Verify

```bash
kubectl get configmap app-config
```

```bash
kubectl describe configmap app-config
```

To see the YAML:

```bash
kubectl get configmap app-config -o yaml
```

Example:

```yaml
apiVersion: v1
data:
  APP_DEBUG: "false"
  APP_ENV: production
  APP_PORT: "8080"
kind: ConfigMap
metadata:
  name: app-config
```

Notice that values such as `false` and `8080` appear as strings.

---

# Task 2: Create a ConfigMap from a File

ConfigMaps can also store complete configuration files.

For this task, an Nginx configuration file was created:

```text
default.conf
```

Contents:

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /health {
        default_type text/plain;
        return 200 "healthy\n";
    }
}
```

This configuration creates a `/health` endpoint that returns:

```text
healthy
```

## Create the ConfigMap

```bash
kubectl create configmap nginx-config \
  --from-file=default.conf=default.conf
```

Expected:

```text
configmap/nginx-config created
```

## Understanding `--from-file`

The syntax is:

```text
--from-file=<key>=<file>
```

In this example:

```bash
--from-file=default.conf=default.conf
```

means:

```text
                ConfigMap key
                     ↓
--from-file=default.conf=default.conf
                              ↑
                         local file
```

The contents of the local `default.conf` file are stored under the `default.conf` key.

The ConfigMap therefore looks conceptually like:

```text
nginx-config
└── default.conf
    └── Nginx configuration
```

## Verify

```bash
kubectl get configmap nginx-config -o yaml
```

The file contents should appear under:

```yaml
data:
  default.conf: |
```

Example:

```yaml
data:
  default.conf: |
    server {
        listen 80;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /health {
            default_type text/plain;
            return 200 "healthy\n";
        }
    }
```

## Important point

The key name matters when the ConfigMap is later mounted as a volume.

Because the key is:

```text
default.conf
```

Kubernetes can create a file named:

```text
default.conf
```

inside the mounted directory.

---

# Task 3: Use ConfigMaps in a Pod

ConfigMaps can be consumed in different ways.

For simple key-value settings, environment variables are useful.

For complete configuration files, volume mounts are more appropriate.

---

## Part 3.1: ConfigMap as Environment Variables

Create:

```text
app-config-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-config-pod
spec:
  containers:
    - name: app
      image: busybox:latest
      command:
        - sh
        - -c
        - |
          echo "APP_ENV=$APP_ENV"
          echo "APP_DEBUG=$APP_DEBUG"
          echo "APP_PORT=$APP_PORT"
      envFrom:
        - configMapRef:
            name: app-config
```

Apply:

```bash
kubectl apply -f app-config-pod.yaml
```

Verify:

```bash
kubectl logs app-config-pod
```

Expected:

```text
APP_ENV=production
APP_DEBUG=false
APP_PORT=8080
```

## How `envFrom` works

This:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

means:

> Take all keys from the `app-config` ConfigMap and create environment variables with those names.

So:

```text
ConfigMap
    │
    ├── APP_ENV=production
    ├── APP_DEBUG=false
    └── APP_PORT=8080
            │
            ▼
Container environment
    │
    ├── APP_ENV=production
    ├── APP_DEBUG=false
    └── APP_PORT=8080
```

### Why did this Pod eventually show `CrashLoopBackOff`?

The BusyBox container was only running commands that printed the variables.

After printing:

```text
APP_ENV=production
APP_DEBUG=false
APP_PORT=8080
```

the command finished.

The main container process therefore exited.

Kubernetes then tried to restart it, causing repeated exits and eventually:

```text
CrashLoopBackOff
```

This does **not** mean the ConfigMap failed.

The logs proved that the ConfigMap was successfully injected.

For a one-time demonstration, seeing the values in the logs is enough.

---

# Part 3.2: ConfigMap as a Volume

For a complete configuration file, use a volume mount.

Create:

```text
nginx-config-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-config-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: nginx-config-volume
      configMap:
        name: nginx-config
```

Apply:

```bash
kubectl apply -f nginx-config-pod.yaml
```

Check:

```bash
kubectl get pods
```

The Nginx Pod should be:

```text
nginx-config-pod    1/1    Running
```

## How the volume mount works

The ConfigMap contains:

```text
nginx-config
└── default.conf
```

The Pod mounts the ConfigMap at:

```text
/etc/nginx/conf.d
```

Therefore Kubernetes creates:

```text
/etc/nginx/conf.d/default.conf
```

inside the container.

The flow is:

```text
ConfigMap
nginx-config
     │
     ▼
ConfigMap volume
     │
     ▼
/etc/nginx/conf.d
     │
     └── default.conf
```

## Verify the mounted file

```bash
kubectl exec nginx-config-pod -- cat /etc/nginx/conf.d/default.conf
```

The custom Nginx configuration should be displayed.

## Test the `/health` endpoint

```bash
kubectl exec nginx-config-pod -- curl -s http://localhost/health
```

Expected:

```text
healthy
```

This proves that:

1. The ConfigMap was created correctly.
2. The ConfigMap was mounted into the Pod.
3. The `default.conf` file was created.
4. Nginx loaded the configuration.
5. The `/health` endpoint works.

---

# ConfigMap: Environment Variable vs Volume

This is one of the most important concepts from this task.

## Environment variable

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

Best for:

```text
APP_ENV
APP_PORT
APP_DEBUG
FEATURE_FLAG
```

Think:

```text
ConfigMap
    ↓
Environment variables
    ↓
Application
```

## Volume mount

```yaml
volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

Best for:

```text
nginx.conf
application.properties
config.yaml
settings.json
```

Think:

```text
ConfigMap
    ↓
Volume
    ↓
Configuration file
    ↓
Application
```

### Easy rule

> **Simple key-value settings → environment variables**

> **Complete configuration files → volume mounts**

---

# Task 4: Create a Secret

Secrets are used for sensitive configuration such as:

* Passwords
* Usernames
* API tokens
* Credentials

Create the Secret:

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD='s3cureP@ssw0rd'
```

Expected:

```text
secret/db-credentials created
```

The Secret contains:

```text
db-credentials
├── DB_USER
└── DB_PASSWORD
```

---

# Inspect the Secret

Run:

```bash
kubectl get secret db-credentials -o yaml
```

The values under `data` will be Base64 encoded.

Example:

```yaml
apiVersion: v1
data:
  DB_PASSWORD: <base64-value>
  DB_USER: <base64-value>
kind: Secret
metadata:
  name: db-credentials
type: Opaque
```

## Decode a value

Take the Base64 value and run:

```bash
echo '<base64-value>' | base64 --decode
```

For example, decoding the password returns:

```text
s3cureP@ssw0rd
```

## Important: Base64 is NOT encryption

This is very important.

```text
Base64 ≠ Encryption
```

Base64 is only an encoding format.

Anyone who has permission to read the Secret can decode the value.

The security of Kubernetes Secrets comes from mechanisms such as:

* RBAC controlling who can access Secrets
* Encryption at rest when configured
* Restricted access to the Kubernetes API
* Kubernetes' handling of Secret data in memory and mounted Secret volumes

Therefore:

> A Kubernetes Secret should not be considered secure simply because its value appears as Base64.

---

# Task 5: Use Secrets in a Pod

A Secret can also be consumed as:

1. An environment variable
2. A mounted file

---

## Create the Pod

Create:

```text
secret-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: busybox:latest
      command:
        - sh
        - -c
        - |
          echo "DB_USER=$DB_USER"
          sleep 3600
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
      volumeMounts:
        - name: db-credentials-volume
          mountPath: /etc/db-credentials
          readOnly: true
  volumes:
    - name: db-credentials-volume
      secret:
        secretName: db-credentials
```

Apply:

```bash
kubectl apply -f secret-pod.yaml
```

Check:

```bash
kubectl get pod secret-pod
```

Expected:

```text
secret-pod    1/1    Running
```

---

# Secret as an Environment Variable

This section:

```yaml
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: DB_USER
```

means:

> Take the `DB_USER` key from the `db-credentials` Secret and provide its value as the `DB_USER` environment variable.

Verify:

```bash
kubectl exec secret-pod -- printenv DB_USER
```

Expected:

```text
admin
```

The application receives:

```text
admin
```

not the Base64 representation.

---

# Secret as a Volume

The Secret is also mounted here:

```yaml
volumeMounts:
  - name: db-credentials-volume
    mountPath: /etc/db-credentials
    readOnly: true
```

The Secret volume is defined here:

```yaml
volumes:
  - name: db-credentials-volume
    secret:
      secretName: db-credentials
```

Because the Secret contains:

```text
DB_USER
DB_PASSWORD
```

Kubernetes creates:

```text
/etc/db-credentials/
├── DB_USER
└── DB_PASSWORD
```

Each Secret key becomes a file.

---

# Verify the Secret Files

List the files:

```bash
kubectl exec secret-pod -- ls /etc/db-credentials
```

Expected:

```text
DB_PASSWORD
DB_USER
```

Read the username:

```bash
kubectl exec secret-pod -- cat /etc/db-credentials/DB_USER
```

Output:

```text
admin
```

Read the password:

```bash
kubectl exec secret-pod -- cat /etc/db-credentials/DB_PASSWORD
```

Output:

```text
s3cureP@ssw0rd
```

## Important: The mounted files are plaintext

This is an important distinction.

When viewing the Secret through the Kubernetes API:

```text
kubectl get secret ... -o yaml
        ↓
Base64 encoded
```

When mounted into a Pod:

```text
Secret
   ↓
Volume
   ↓
File
   ↓
Plaintext value
```

Therefore:

> **Secret data is Base64 encoded in the Kubernetes API representation, but mounted Secret files contain the decoded plaintext value.**

The same applies when the Secret is injected through `secretKeyRef`.

---

# Task 6: Update a ConfigMap and Observe Propagation

This task demonstrates an important difference between:

* ConfigMaps mounted as volumes
* ConfigMaps used as environment variables

---

## Create the ConfigMap

```bash
kubectl create configmap live-config \
  --from-literal=message=hello
```

Verify:

```bash
kubectl get configmap live-config -o yaml
```

The data should contain:

```yaml
data:
  message: hello
```

---

# Create a Pod that Reads the Mounted File

Create:

```text
live-config-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: live-config-pod
spec:
  containers:
    - name: app
      image: busybox:latest
      command:
        - sh
        - -c
        - |
          while true; do
            echo "Message: $(cat /etc/config/message)"
            sleep 5
          done
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: live-config
```

Apply:

```bash
kubectl apply -f live-config-pod.yaml
```

Check:

```bash
kubectl get pod live-config-pod
```

Then watch the logs:

```bash
kubectl logs -f live-config-pod
```

Initially:

```text
Message: hello
Message: hello
Message: hello
```

---

# Update the ConfigMap

Run:

```bash
kubectl patch configmap live-config \
  --type merge \
  -p '{"data":{"message":"world"}}'
```

Expected:

```text
configmap/live-config patched
```

Wait around 30-60 seconds.

The logs should eventually change to:

```text
Message: world
Message: world
Message: world
```

No Pod restart is required.

---

# Why did the value change?

The ConfigMap was mounted as a **volume**.

Kubernetes periodically refreshes the mounted ConfigMap data.

The flow is:

```text
ConfigMap
message=hello
     ↓
Mounted file
/etc/config/message
     ↓
hello
```

After updating the ConfigMap:

```text
ConfigMap
message=world
     ↓
Kubernetes refreshes mounted volume
     ↓
/etc/config/message
     ↓
world
```

The Pod itself doesn't need to restart.

---

# Environment Variables Behave Differently

Environment variables are created when the container starts.

For example:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

The container starts with:

```text
APP_ENV=production
```

If the ConfigMap is later changed to:

```text
APP_ENV=staging
```

the existing container environment does not automatically change.

The container needs to be restarted/recreated to receive the new environment variable value.

Therefore:

```text
ConfigMap as volume
→ mounted file can be updated automatically


ConfigMap as environment variable
→ value is set when the container starts
→ existing container does not automatically receive updates
```

## Easy way to remember

```text
ENV VARIABLE
"Give my application this value when it starts."

VOLUME
"Give my application this configuration file."
```

---

# Task 7: Clean Up

Delete the Pods:

```bash
kubectl delete pod \
  app-config-pod \
  nginx-config-pod \
  secret-pod \
  live-config-pod
```

Delete the ConfigMaps:

```bash
kubectl delete configmap \
  app-config \
  nginx-config \
  live-config
```

Delete the Secret:

```bash
kubectl delete secret db-credentials
```

Verify:

```bash
kubectl get pods
kubectl get configmaps
kubectl get secrets
```

Do not delete Kubernetes-managed resources that were not created for this exercise.

---

# Final Mental Model

The entire Day 54 can be remembered with this diagram:

```text
                    Kubernetes Configuration
                             │
               ┌─────────────┴─────────────┐
               │                           │
          ConfigMap                     Secret
        Non-sensitive                 Sensitive
               │                           │
       ┌───────┴───────┐           ┌───────┴───────┐
       │               │           │               │
       ▼               ▼           ▼               ▼
 Environment        Volume      Environment      Volume
 Variables          Mount       Variables        Mount
       │               │           │               │
       ▼               ▼           ▼               ▼
 APP_ENV          config file    DB_USER       DB_PASSWORD
 APP_PORT         nginx.conf     API_TOKEN      credentials
```

## Key Commands

### ConfigMap from values

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080
```

### ConfigMap from a file

```bash
kubectl create configmap nginx-config \
  --from-file=default.conf=default.conf
```

### Secret from values

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD='s3cureP@ssw0rd'
```

### Inspect a ConfigMap

```bash
kubectl get configmap app-config -o yaml
```

### Inspect a Secret

```bash
kubectl get secret db-credentials -o yaml
```

### Decode Base64

```bash
echo '<base64-value>' | base64 --decode
```

---

# Key Takeaways

1. **ConfigMaps store non-sensitive configuration.**

2. **Secrets store sensitive configuration.**

3. `--from-literal` puts values directly into the command.

4. `--from-file` takes configuration from a file.

5. `envFrom` can inject all ConfigMap keys as environment variables.

6. `secretKeyRef` can inject one Secret key as an environment variable.

7. ConfigMaps and Secrets can both be mounted as volumes.

8. When a ConfigMap or Secret is mounted as a volume, each key can become a file.

9. Secret values shown through the Kubernetes API are Base64 encoded.

10. **Base64 is encoding, not encryption.**

11. Mounted Secret files contain the decoded plaintext value.

12. ConfigMap values mounted as volumes can be updated without restarting the Pod.

13. Environment variables from ConfigMaps or Secrets are set when the container starts and do not automatically update inside an existing container.

14. Use **environment variables for simple key-value configuration** and **volume mounts for complete configuration files**.
