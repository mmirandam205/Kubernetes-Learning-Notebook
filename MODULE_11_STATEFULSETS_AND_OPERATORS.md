# Module 11: StatefulSets and Operators

## Stateless vs Stateful Applications
A stateless application is one where each request is handled independently and no information from previous requests needs to be retained by the instance handling it. Web servers, REST API services, and most microservices are stateless. Any replica of the application is interchangeable with any other. If a Pod dies and is replaced, the new Pod picks up exactly where the old one left off from the client's perspective — because state lives elsewhere, typically in a database or cache.

A stateful application, by contrast, requires that its state be preserved across restarts and tied to a specific instance. Databases like MySQL and PostgreSQL, message queues like Kafka and RabbitMQ, and coordination systems like ZooKeeper and etcd are stateful. Each instance may hold a unique portion of the data, maintain a unique identity in a cluster protocol, or require a specific ordered startup sequence.

Kubernetes Deployments are designed for stateless workloads. Every Pod created by a Deployment is identical and interchangeable. Pods receive random names, may be scheduled to any node, and share a single PersistentVolumeClaim if one is attached. This is fundamentally incompatible with stateful workloads, which need stable identities, ordered operations, and per-instance storage.

## StatefulSets

### How StatefulSets Differ from Deployments
A StatefulSet is a Kubernetes workload resource designed specifically for stateful applications. Unlike Deployments, StatefulSets provide three guarantees that stateful workloads depend on:

**Stable, unique network identities**: Each Pod in a StatefulSet gets a predictable, persistent hostname based on its ordinal index. If the StatefulSet is named `mysql` and has three replicas, the Pods will be named `mysql-0`, `mysql-1`, and `mysql-2`. These names persist across rescheduling. If `mysql-1` is deleted and recreated, the new Pod still gets the name `mysql-1`, not a random string.

**Ordered deployment and scaling**: Pods are created in order from 0 to N-1, and each Pod must be Running and Ready before the next one starts. Scaling down happens in reverse order, from N-1 to 0. This matters for applications like Kafka or ZooKeeper where a specific bootstrap order is required.

**Stable persistent storage via volumeClaimTemplates**: Each Pod gets its own dedicated PersistentVolumeClaim, created from a template defined in the StatefulSet spec. When a Pod is rescheduled, it reattaches to the same PVC. The storage follows the identity, not the node.

### Headless Services and DNS Resolution
StatefulSets work in conjunction with a headless Service — a Service with `clusterIP: None`. A headless Service does not provide load balancing. Instead, it creates individual DNS A records for each Pod in the StatefulSet. The DNS format is:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For example, `mysql-0.mysql.default.svc.cluster.local` resolves directly to the IP address of the `mysql-0` Pod. This allows other services or Pods within the same StatefulSet to address specific replicas by name, which is essential for leader election and replication configuration in databases.

### StatefulSet YAML Example
The following example creates a StatefulSet for a three-replica MySQL cluster, including a headless Service and per-Pod persistent storage:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

This StatefulSet creates three Pods (`mysql-0`, `mysql-1`, `mysql-2`), each with its own 10Gi PVC (`data-mysql-0`, `data-mysql-1`, `data-mysql-2`). The headless Service enables DNS-based direct addressing of individual Pods.

### Managing StatefulSets
```bash
kubectl get statefulsets
kubectl describe statefulset mysql
kubectl scale statefulset mysql --replicas=5
```

Common use cases for StatefulSets include MySQL, PostgreSQL, Kafka, ZooKeeper, Elasticsearch, and Redis Sentinel clusters. Any workload that requires stable identity and per-instance persistence is a candidate.

## DaemonSets
A DaemonSet ensures that exactly one Pod runs on every node in the cluster, or on a selected subset of nodes that match a node selector. When a new node joins the cluster, the DaemonSet automatically schedules a Pod onto it. When a node is removed, the Pod is garbage collected.

DaemonSets are the right tool for infrastructure-level services that need to run on every node:

- Log collectors such as Fluentd or Fluent Bit, which read log files from the node's filesystem
- Node monitoring agents such as Prometheus node-exporter or Datadog Agent
- CNI plugins and network proxies that configure per-node networking
- Security agents that inspect node-level activity

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

## Jobs and CronJobs

### Jobs
A Job creates one or more Pods and tracks their successful completion. Unlike Deployments, Jobs are designed for run-to-completion workloads — tasks that have a defined end state. When the required number of Pods complete successfully, the Job is marked as complete.

Common use cases for Jobs include database schema migrations, batch data processing, report generation, and one-off administrative tasks.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: my-app:latest
        command: ["./migrate.sh"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

The `completions` field specifies how many successful Pod completions are needed. The `parallelism` field controls how many Pods run in parallel. The `backoffLimit` field limits how many times a failed Pod is retried before the Job is marked as failed.

### CronJobs
A CronJob creates Jobs on a schedule defined by a standard cron expression. Each scheduled run creates a new Job object, which in turn creates Pods.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: reporter
            image: report-generator:latest
            command: ["./generate-report.sh"]
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

This CronJob runs at 2:00 AM every night. The `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` fields control how many completed Job objects are retained for inspection.

```bash
kubectl get jobs
kubectl get cronjobs
```

## Operators Pattern

### What is an Operator?
An Operator is a software extension that uses Kubernetes custom resources to manage an application and its components. The term was coined by CoreOS (now part of Red Hat) to describe the practice of encoding operational knowledge — the kind of knowledge a human operator would use to deploy, scale, back up, and upgrade a complex application — into software that runs inside the Kubernetes cluster.

An Operator consists of two parts:

- **Custom Resource Definition (CRD)**: Extends the Kubernetes API with a new resource type. For example, a PostgreSQL Operator might define a `PostgresCluster` resource. You interact with it like any other Kubernetes resource using `kubectl get postgresclusters`.
- **Controller**: A running program (usually a Deployment) that watches for changes to the custom resource and takes action to reconcile the actual state of the application with the desired state described in the resource.

### The Operator Pattern
The Operator pattern is built on Kubernetes' reconciliation loop. The controller continuously compares the desired state expressed in the custom resource against the actual state of the managed application. When they differ — because a new version was specified, a replica count changed, or a backup was requested — the controller takes action to close the gap.

This codifies operational knowledge that would otherwise require a human operator to perform manual steps. A PostgreSQL Operator knows how to set up streaming replication between a primary and replicas, how to perform a failover when the primary crashes, how to trigger a backup to object storage, and how to apply a minor version upgrade with minimal downtime.

### Examples of Well-Known Operators
- **PostgreSQL Operator** (CloudNativePG, Crunchy Data, Zalando): Manages PostgreSQL clusters with automatic failover, backup, and restore
- **Prometheus Operator**: Manages Prometheus, Alertmanager, and scrape configuration via `ServiceMonitor` and `PrometheusRule` custom resources
- **Kafka Operator** (Strimzi): Manages Kafka clusters, topics, and users on Kubernetes
- **cert-manager**: Manages TLS certificate issuance and renewal from Let's Encrypt and other CAs

### Finding and Building Operators
OperatorHub.io is the community registry for Kubernetes Operators. It hosts hundreds of Operators for databases, monitoring tools, cloud services, and more, many of them certified by Red Hat or their vendors.

Operators are typically built using the Operator SDK or Kubebuilder, both of which provide scaffolding and libraries for writing controllers in Go. The Operator SDK also supports writing Operators in Ansible or Helm for simpler use cases.

## When to Use StatefulSets vs Operators
A StatefulSet is the right tool for straightforward stateful workloads where you are willing to handle the operational aspects manually — initial configuration, backups, failover — or where the application is simple enough that these concerns are minimal. A single-instance PostgreSQL database for a development environment, or a small Kafka cluster for a non-critical service, are reasonable StatefulSet candidates.

An Operator is the right tool when the operational complexity of the application justifies automating it. Production databases requiring automatic failover, backup scheduling, and zero-downtime upgrades benefit enormously from an Operator. The tradeoff is that Operators introduce an additional system to trust and maintain.

## Best Practices
Always use `volumeClaimTemplates` in StatefulSets rather than `hostPath` volumes. Host path volumes tie a Pod to a specific node and prevent rescheduling. PVCs allow the storage to persist independently of the Pod's location.

Set Pod Disruption Budgets for stateful workloads to prevent the Cluster Autoscaler or node maintenance operations from evicting too many Pods at once. A database cluster that loses quorum due to simultaneous Pod evictions can become unavailable or even lose data.

Use well-maintained community Operators from OperatorHub rather than writing your own for common databases and messaging systems. These Operators incorporate years of operational experience and are actively maintained.

Never allow automated tools to delete the PVCs associated with a StatefulSet. When deleting a StatefulSet, use `--cascade=orphan` to keep the Pods (and their PVCs) running, then clean up manually after confirming the data has been safely backed up or migrated:

```bash
kubectl delete statefulset mysql --cascade=orphan
```
