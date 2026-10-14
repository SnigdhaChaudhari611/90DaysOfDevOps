# Day 55: Kubernetes Persistent Volumes

## Why Containers Need Persistent Storage

Containers are designed to be replaceable. Data written only to a container's writable filesystem can be lost when that container is removed and recreated.

Kubernetes offers storage resources that let applications keep data beyond a container's lifecycle.

- `emptyDir` provides temporary storage shared by containers in the same Pod. It survives a container restart while the Pod remains, but is deleted when the Pod is removed.
- Persistent storage is appropriate for data that must survive Pod deletion and recreation, such as application files or database data.

The storage's durability also depends on the volume type and where its backing data lives. For example, a `hostPath` volume uses a directory on a particular node and does not automatically make that data available on another node.

## PersistentVolume (PV) and PersistentVolumeClaim (PVC)

### PersistentVolume (PV)

A **PersistentVolume** is a cluster resource representing storage made available to Kubernetes. It describes details such as capacity, access modes, storage source, and reclaim policy.

A PV can be created by an administrator or provisioned dynamically by a StorageClass.

### PersistentVolumeClaim (PVC)

A **PersistentVolumeClaim** is a request for storage made by a user or application. It specifies requirements such as requested capacity, access modes, and optionally a StorageClass.

### How They Relate

1. A workload requests storage through a PVC.
2. Kubernetes finds a suitable PV or provisions one dynamically through a StorageClass.
3. The PVC binds to a PV that meets its requirements.
4. A Pod references the PVC and mounts the volume at a path inside its container.

A PVC is the request; the PV represents the storage. A Pod normally uses the PVC rather than referencing the PV directly.

## Static vs Dynamic Provisioning

### Static Provisioning

An administrator creates the PV before a PVC requests it.

Example from this lab:

- PV: `demo-pv`
- Capacity: `1Gi`
- Access mode: `ReadWriteOnce`
- Reclaim policy: `Retain`
- Storage source: `hostPath` at `/tmp/k8s-pv-data`

The PVC `demo-pvc` requested `500Mi` and bound to `demo-pv`. Its `storageClassName: ""` setting allowed it to bind to this manually created PV without requesting the default StorageClass.

### Dynamic Provisioning

Kubernetes provisions storage automatically when a PVC requests a StorageClass and a suitable volume is needed.

In this lab, the `standard` StorageClass used the `rancher.io/local-path` provisioner. The `dynamic-pvc` claim requested `500Mi`; when a Pod used the claim, Kubernetes dynamically created a PV with a name similar to `pvc-...`.

The StorageClass used these settings:

- Provisioner: `rancher.io/local-path`
- Reclaim policy: `Delete`
- Volume binding mode: `WaitForFirstConsumer`
- Volume expansion: disabled

`WaitForFirstConsumer` delays binding or provisioning until a Pod using the PVC is available for scheduling. This helps the provisioner choose storage in relation to the Pod's scheduled node.

Local-path storage is node-local; it is not equivalent to network-attached storage such as Amazon EBS.

## Access Modes

Access modes describe how a volume may be mounted by nodes or workloads. They do not, by themselves, guarantee that an application can safely write concurrently.

| Access mode | Meaning |
| --- | --- |
| `ReadWriteOnce` (RWO) | The volume can be mounted read-write by a single node. Multiple Pods on that same node may be able to use it, depending on the volume and workload setup. |
| `ReadOnlyMany` (ROX) | The volume can be mounted read-only by many nodes, if the storage type supports it. |
| `ReadWriteMany` (RWX) | The volume can be mounted read-write by many nodes, if the storage type supports it. |
| `ReadWriteOncePod` (RWOP) | The volume can be mounted read-write by a single Pod, when supported by the CSI driver and Kubernetes version. |

Not every storage backend supports every access mode. The mode requested by a PVC must be supported by the volume.

## Reclaim Policies

A PV's reclaim policy determines what Kubernetes does with the volume after its PVC is deleted.

| Policy | Behavior |
| --- | --- |
| `Retain` | Keeps the PV and underlying storage for manual recovery or cleanup. The PV typically moves to `Released` and is not automatically available for a new claim. |
| `Delete` | The PV and its backing storage are eligible for deletion by the provisioner when the claim is deleted and cleanup completes. |
| `Recycle` | Legacy basic scrub behavior. Deprecated and not recommended for new workloads. |

### What I Observed in This Lab

- The `emptyDir` file survived a container restart but disappeared when its Pod was deleted and recreated.
- Data stored through `demo-pvc` remained available after deleting and recreating the Pod that mounted it.
- `demo-pv` was manually created with the `Retain` policy. After deleting `demo-pvc`, the PV remained with status `Released` until I deleted it manually.
- The dynamically provisioned PV used the `Delete` policy and disappeared after `dynamic-pvc` was deleted and cleanup completed.

**Important:** Deleting a PV object does not necessarily remove data from a manually managed storage path. Likewise, exact backing-storage cleanup depends on the volume type and provisioner.

## Useful Commands

```bash
# Inspect Pods, claims, volumes, and StorageClasses
kubectl get pods
kubectl get pvc
kubectl get pv
kubectl get storageclass

# Inspect details
kubectl describe pvc demo-pvc
kubectl describe pv demo-pv
kubectl describe storageclass standard

# Check data mounted in a Pod
kubectl exec dynamic-pod -- cat /data/message.txt
```

## Key Takeaways

- Container filesystems are not a reliable place for data that must outlive a container.
- A PVC requests storage; a PV represents the storage that satisfies the request.
- Static provisioning uses an existing PV; dynamic provisioning creates storage through a StorageClass.
- Access modes describe supported mount access, and actual capabilities depend on the storage backend.
- Reclaim policies control how volumes are handled after a PVC is deleted.
- `hostPath` and local-path storage are node-local and should not be mistaken for highly available shared storage.
