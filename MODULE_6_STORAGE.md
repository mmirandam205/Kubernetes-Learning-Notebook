# Module 6: Storage and Persistence

## Why Storage Matters in Kubernetes
Containers are ephemeral by nature — when a container restarts, all data written to its filesystem is lost. Kubernetes provides a robust storage system to handle persistent data needs for stateful applications like databases, message queues, and file storage services. Understanding volumes, PersistentVolumes, and StorageClasses is essential for running real-world workloads reliably.

## Volumes
A Volume in Kubernetes is a directory accessible to containers in a Pod. Unlike a container's local filesystem, a Volume's lifetime is tied to the Pod, not the container — so data survives container restarts within the same Pod. Kubernetes supports many volume types depending on where the data lives.

Common volume types:
- **emptyDir**: Created when a Pod is assigned to a node, deleted when the Pod is removed. Useful for temporary scratch space or sharing data between containers in the same Pod.
- **hostPath**: Mounts a file or directory from the host node's filesystem into the Pod. Useful for development but not recommended for production.
- **configMap / secret**: Mounts ConfigMap or Secret data as files inside a container.
- **nfs**: Mounts an NFS share into the Pod, allowing multiple Pods to share the same data.
- **persistentVolumeClaim**: The most common production pattern — binds a Pod to a PersistentVolumeClaim.

## PersistentVolumes (PV)
A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically by a StorageClass. It is a cluster-level resource, independent of any individual Pod. PVs have a lifecycle separate from the Pods that use them, meaning data persists even after a Pod is deleted.

Key PV attributes:
- **Capacity**: The amount of storage (e.g., 10Gi)
- **Access Modes**:
  - ReadWriteOnce (RWO) — mounted as read-write by a single node
  - ReadOnlyMany (ROX) — mounted as read-only by many nodes
  - ReadWriteMany (RWX) — mounted as read-write by many nodes
- **Reclaim Policy**:
  - Retain — manual cleanup required
  - Delete — volume is deleted automatically
  - Recycle — basic scrub (deprecated)

## PersistentVolumeClaims (PVC)
A PersistentVolumeClaim (PVC) is a request for storage by a user or application. It specifies the size and access mode required, and Kubernetes binds it to a matching PersistentVolume. PVCs decouple the application from the underlying storage infrastructure.

Key commands:
- kubectl get pv — list all PersistentVolumes in the cluster
- kubectl get pvc — list all PersistentVolumeClaims in the namespace
- kubectl describe pvc name — show binding status and details
- kubectl delete pvc name — delete a claim

## StorageClasses
A StorageClass provides a way to describe the class of storage available in a cluster. Different classes might map to different quality-of-service levels, backup policies, or storage backends. StorageClasses enable dynamic provisioning — automatically creating a PV when a PVC is submitted.

Key StorageClass concepts:
- **provisioner**: The plugin that creates the storage (e.g., kubernetes.io/aws-ebs, kubernetes.io/gce-pd)
- **reclaimPolicy**: Default reclaim policy for dynamically provisioned PVs
- **volumeBindingMode**: Immediate (default) or WaitForFirstConsumer
- kubectl get storageclass — list available storage classes

## StatefulSets and Persistent Storage
StatefulSets are designed for stateful applications that require stable, persistent storage per Pod. Each Pod in a StatefulSet gets its own PVC via a volumeClaimTemplate, ensuring that each replica has dedicated storage that is not shared with other replicas. This is the standard pattern for running databases like PostgreSQL, MySQL, and MongoDB on Kubernetes.

StatefulSet storage characteristics:
- Each Pod gets a unique, stable PVC (e.g., data-mysql-0, data-mysql-1)
- PVCs are not deleted when the StatefulSet is scaled down
- Pods are created and deleted in order (0, 1, 2...) for predictable startup/shutdown

## Storage Best Practices
- Always use PVCs rather than hostPath in production
- Choose the correct access mode for your workload (most databases need RWO)
- Use StorageClasses with WaitForFirstConsumer in multi-zone clusters
- Set resource requests on PVCs to avoid over-provisioning
- Regularly back up persistent data using tools like Velero
- Monitor PV usage to avoid running out of disk space unexpectedly

## Why Storage is Critical
Without proper storage management, stateful applications cannot run reliably on Kubernetes. A database that loses its data on every restart, or a file service that cannot share data between replicas, will not function correctly in production. Mastering Kubernetes storage patterns is a key step toward running enterprise-grade workloads.