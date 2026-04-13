# Module 10: Advanced Scheduling

## How the Kubernetes Scheduler Works
When a Pod is created, it starts in the Pending state. The Kubernetes scheduler watches for unscheduled Pods and assigns each one to a node by writing the node name into the Pod's spec. This process happens in two phases: filtering and scoring.

During filtering, the scheduler evaluates every node in the cluster and eliminates those that cannot run the Pod. Reasons a node might be filtered out include insufficient CPU or memory, a node selector that does not match the node's labels, a taint the Pod cannot tolerate, or a volume that is not available in that zone.

During scoring, the scheduler ranks the remaining candidate nodes using a set of priority functions. These consider factors like resource balance (spreading load evenly), image locality (preferring nodes that already have the container image cached), and affinity preferences. The node with the highest score is selected.

After scoring, the scheduler writes the chosen node name into the Pod spec — a process called binding. The kubelet on that node then detects the new Pod assignment and starts it.

Kubernetes ships with a default scheduler that handles the vast majority of use cases. For specialized workloads, you can deploy a custom scheduler alongside the default one and instruct specific Pods to use it via the `schedulerName` field in the Pod spec.

## Node Selectors
The simplest way to constrain a Pod to a specific subset of nodes is to use `nodeSelector`. You first label the nodes you want to target, then reference those labels in the Pod spec.

Label a node:

```bash
kubectl label nodes worker-node-1 disktype=ssd
```

Then specify the selector in the Pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-workload
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: my-app:latest
```

This Pod will only be scheduled on nodes that have the label `disktype=ssd`. Node selectors are simple and predictable, but they are inflexible. They only support exact-match equality and cannot express preferences — either the label matches or the Pod will not be scheduled. For more nuanced requirements, use node affinity instead.

## Node Affinity and Anti-Affinity
Node affinity is a more expressive replacement for `nodeSelector`. It supports a richer set of operators (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`) and distinguishes between hard requirements and soft preferences.

There are two affinity types:

- `requiredDuringSchedulingIgnoredDuringExecution`: A hard requirement. The Pod will not be scheduled on a node that does not satisfy the rule. This is equivalent to `nodeSelector` but with richer expression.
- `preferredDuringSchedulingIgnoredDuringExecution`: A soft preference. The scheduler will try to satisfy the rule but will fall back to other nodes if no suitable nodes exist. Each preference has a weight between 1 and 100 that influences the scoring phase.

The "IgnoredDuringExecution" suffix means that if a node's labels change after the Pod is already running, the running Pod is not evicted. Future versions of Kubernetes may introduce `RequiredDuringExecution` variants.

Example — require the Pod to run on a GPU node, and prefer a node in zone `us-east-1a`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values:
            - nvidia-tesla-v100
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: gpu-app
    image: my-gpu-app:latest
```

## Pod Affinity and Anti-Affinity
While node affinity controls which nodes a Pod can land on, pod affinity and anti-affinity control how Pods are positioned relative to other Pods. This is useful for co-locating services that communicate frequently (affinity) or for spreading replicas apart for fault tolerance (anti-affinity).

Pod affinity and anti-affinity rules are evaluated at scheduling time. They use a `topologyKey` to define the scope of colocation. The `kubernetes.io/hostname` topology key restricts evaluation to the same node, while `topology.kubernetes.io/zone` evaluates at the availability zone level.

Example — co-locate a cache Pod on the same node as its primary application:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache
  labels:
    app: cache
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: primary
        topologyKey: kubernetes.io/hostname
  containers:
  - name: cache
    image: redis:7
```

Example — spread web server replicas across different nodes to avoid a single point of failure:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web-server
            topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx:1.25
```

This ensures that no two web-server Pods land on the same node. If fewer nodes are available than replicas, some Pods will remain unschedulable.

## Taints and Tolerations
Taints are placed on nodes to repel Pods. A Pod will not be scheduled on a tainted node unless it explicitly declares a toleration for that taint. This mechanism is the opposite of node affinity: instead of Pods choosing nodes, nodes reject Pods.

Apply a taint to a node:

```bash
kubectl taint nodes worker-node-2 dedicated=gpu:NoSchedule
```

This prevents any Pod that does not tolerate `dedicated=gpu` from being scheduled on `worker-node-2`.

To allow a Pod onto a tainted node, add a matching toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: trainer
    image: my-gpu-trainer:latest
```

Taint effects determine the severity of the restriction:

- **NoSchedule**: New Pods without a matching toleration will not be scheduled on the node. Existing Pods are unaffected.
- **PreferNoSchedule**: The scheduler tries to avoid placing Pods on the node but may do so if no other node is available.
- **NoExecute**: New Pods are not scheduled and existing Pods that do not tolerate the taint are evicted.

Common use cases for taints include dedicating nodes to specific workloads (GPU training, high-memory jobs), isolating the control plane (master nodes are tainted with `node-role.kubernetes.io/control-plane:NoSchedule` by default), and implementing maintenance windows by tainting a node with `NoExecute` before draining it.

To remove a taint, append a minus sign:

```bash
kubectl taint nodes worker-node-2 dedicated=gpu:NoSchedule-
```

## Topology Spread Constraints
Topology spread constraints are a more modern and flexible alternative to pod anti-affinity for distributing Pods evenly across a topology. While anti-affinity can prevent two Pods from sharing a node, topology spread constraints enforce an even distribution across any number of topology domains such as zones, nodes, or racks.

Key fields:

- `maxSkew`: The maximum allowed difference in Pod count between any two topology domains. A `maxSkew` of 1 means no domain can have more than one extra Pod compared to the least-populated domain.
- `topologyKey`: The node label that defines the topology domain (e.g., `topology.kubernetes.io/zone`).
- `whenUnsatisfiable`: What to do when the constraint cannot be satisfied. `DoNotSchedule` treats it as a hard requirement; `ScheduleAnyway` treats it as a preference.

Example — spread Pods evenly across availability zones:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 6
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api-server
      containers:
      - name: api
        image: api-server:latest
```

With three availability zones and six replicas, this constraint ensures exactly two Pods land in each zone. Topology spread constraints are generally preferred over pod anti-affinity for even distribution because they scale well with increasing replica counts and provide clearer semantics.

## Priority and Preemption
Priority classes allow you to define the relative importance of Pods within a cluster. When a cluster is running low on resources, the scheduler can evict (preempt) lower-priority Pods to make room for higher-priority ones.

A PriorityClass resource assigns an integer priority value to a name:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Used for critical production workloads that must not be preempted."
```

Pods reference a PriorityClass by name:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-service
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: critical-app:latest
```

Kubernetes ships with two built-in priority classes: `system-cluster-critical` and `system-node-critical`, used by core system components like CoreDNS and kube-proxy. Avoid giving user workloads priorities that compete with these system classes.

When a high-priority Pod cannot be scheduled due to resource constraints, the scheduler identifies lower-priority Pods whose eviction would free enough resources and evicts them. The evicted Pods return to the Pending state. Use priority classes thoughtfully — overusing high priorities can cause cascading evictions and instability.

## Resource Quotas and LimitRanges in Scheduling Context
Resource quotas set aggregate limits on the total CPU, memory, and object count a namespace can consume. When a Pod is created, the scheduler verifies that its resource requests do not exceed the remaining quota. If the namespace is at capacity, the Pod creation is rejected before scheduling even begins.

LimitRanges set default and maximum resource requests and limits for containers within a namespace. They ensure that Pods without explicit resource specifications still have sensible defaults, which is important for HPA and scheduling to work predictably.

Both mechanisms act as guardrails that shape which Pods can be created and therefore which Pods compete for scheduling.

## Scheduling Best Practices
Use affinity and anti-affinity rules thoughtfully. Overly strict required affinity rules can leave Pods unschedulable if matching nodes are unavailable. Start with preferred rules and only escalate to required rules when the constraint is truly non-negotiable.

Prefer topology spread constraints over pod anti-affinity for even distribution. Anti-affinity rules with `requiredDuringScheduling` become hard to satisfy as replica counts grow; topology spread constraints handle this gracefully.

Label nodes consistently and deliberately. A well-designed labeling scheme covering zone, instance type, and workload class enables expressive scheduling policies without constant changes to node configurations.

Avoid over-constraining Pods with multiple conflicting required rules. Each additional required constraint narrows the set of eligible nodes. A Pod with three conflicting required constraints may never be schedulable even though the cluster has plenty of capacity.

Test scheduling rules in a staging cluster before production. Use `kubectl describe pod` on a Pending Pod to read the scheduler's detailed reason for not placing it, and iterate until the rules produce the desired placement.
