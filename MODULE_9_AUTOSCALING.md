# Module 9: Autoscaling in Kubernetes

## What is Autoscaling?
Autoscaling is the ability of a system to automatically adjust the amount of computational resources it uses based on current demand. In Kubernetes, this means either adding more Pod replicas to handle increased traffic, adjusting the resource allocations of existing Pods, or adding more nodes to the cluster when capacity runs out.

Manual scaling — where an operator logs in and changes replica counts by hand — works fine in stable, predictable environments. But most real workloads are not predictable. Traffic spikes during business hours, batch jobs consume burst capacity, and services experience sudden load from upstream events. Reacting manually introduces human latency and increases the risk of either over-provisioning (wasting money) or under-provisioning (causing degraded performance or outages).

Kubernetes provides three distinct autoscaling mechanisms, each operating at a different level:

- **Horizontal Pod Autoscaler (HPA)**: Increases or decreases the number of Pod replicas for a Deployment, StatefulSet, or ReplicaSet based on observed metrics.
- **Vertical Pod Autoscaler (VPA)**: Adjusts the CPU and memory requests and limits of individual containers rather than changing the number of replicas.
- **Cluster Autoscaler (CA)**: Adds or removes nodes from the underlying cluster when Pods cannot be scheduled due to insufficient capacity or when nodes are underutilized.

These mechanisms are complementary. A well-designed autoscaling strategy typically combines all three.

## Horizontal Pod Autoscaler (HPA)

### How HPA Works
The HPA controller runs inside the Kubernetes control plane and periodically queries the Metrics Server (or a custom metrics adapter) to retrieve resource usage data for the target workload. It then compares the observed metric value against the target threshold you have defined and calculates how many replicas are needed to bring the metric back to the target.

The calculation uses the formula:

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

For example, if you have two replicas each at 90% CPU and the target is 50%, the controller calculates `ceil[2 * (90 / 50)] = ceil[3.6] = 4` replicas. The controller then scales the Deployment to four replicas. Once traffic decreases, the calculation produces a smaller number and the controller scales back down, respecting a stabilization window to avoid thrashing.

HPA requires the Metrics Server to be installed in the cluster. Without it, the HPA controller cannot retrieve resource metrics and will report an unknown state.

### Key HPA Fields
- `minReplicas`: The minimum number of replicas the HPA will scale down to. This prevents the workload from scaling to zero.
- `maxReplicas`: The maximum number of replicas the HPA will ever create. This acts as a cost control ceiling.
- `targetCPUUtilizationPercentage`: The target average CPU utilization across all Pods, expressed as a percentage of the Pod's CPU request.

### Creating an HPA with YAML
The following example creates an HPA for a Deployment named `api-server`, keeping between 2 and 10 replicas and targeting 50% average CPU utilization:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Imperative HPA Creation
You can also create an HPA using the imperative command without writing YAML:

```bash
kubectl autoscale deployment api-server --cpu-percent=50 --min=2 --max=10
```

### Inspecting an HPA
Once an HPA is running you can check its current state with:

```bash
kubectl get hpa
kubectl describe hpa api-server-hpa
```

The `kubectl get hpa` output shows the current replica count, the desired replica count, and the current metric value. The `describe` output provides the full event history, including scaling decisions and any errors fetching metrics.

### Scaling on Custom Metrics
Beyond CPU and memory, HPA supports custom metrics and external metrics through adapters. The Prometheus Adapter translates Prometheus metrics into the Kubernetes custom metrics API, allowing you to scale on application-level signals such as requests per second, queue depth, or error rate.

KEDA (Kubernetes Event-Driven Autoscaling) is a more powerful alternative that supports over 50 event sources out of the box, including AWS SQS, Azure Service Bus, Kafka, Redis, and Prometheus. KEDA extends the HPA concept and can also scale workloads all the way down to zero replicas when there are no events to process, which is particularly useful for batch and event-driven workloads.

## Vertical Pod Autoscaler (VPA)

### What VPA Does
While HPA changes the number of replicas, the Vertical Pod Autoscaler (VPA) changes the resource requests and limits of individual containers. If a container is consistently using more memory than it requested, VPA can increase its memory request so the scheduler places it on a node with sufficient capacity. If a container is consistently over-provisioned, VPA can reduce its requests to improve bin-packing and reduce cost.

VPA is not a built-in Kubernetes component. It must be installed separately from the Kubernetes Autoscaler project on GitHub.

### VPA Modes
VPA operates in one of four modes:

- **Off**: VPA only provides recommendations but does not apply them. Use this mode to get suggestions before enabling automation.
- **Initial**: VPA sets resource requests only when a Pod is first created and does not change running Pods.
- **Recreate**: VPA can evict running Pods to apply updated resource recommendations. The Pod is then recreated with the new requests.
- **Auto**: VPA uses either Recreate or in-place updates (when supported by the cluster) to apply recommendations. This is the most automated mode.

### When to Use VPA vs HPA
Use HPA when your workload scales horizontally — meaning running more copies of the same container handles increased load. This is appropriate for stateless web servers, API services, and worker processes.

Use VPA when your workload is not easily parallelizable but benefits from more or less memory and CPU per instance. Databases, JVM-based applications with heap sizing considerations, and single-instance services are common VPA candidates.

Avoid running HPA and VPA in Auto or Recreate mode simultaneously on the same Deployment targeting CPU, because they will fight each other. If you need both, set VPA to Off or Initial mode for recommendations only, or use VPA exclusively on memory while HPA handles CPU.

## Cluster Autoscaler (CA)

### What the Cluster Autoscaler Does
The Cluster Autoscaler monitors for Pods that cannot be scheduled because no node has sufficient available resources. When it detects unschedulable Pods, it provisions additional nodes from the underlying cloud provider's node group. Conversely, when nodes are consistently underutilized and all their Pods could be moved to other nodes, the Cluster Autoscaler drains and terminates those nodes to reduce cost.

The Cluster Autoscaler integrates directly with cloud provider APIs. On AWS it works with Auto Scaling Groups, on GCP it works with Managed Instance Groups, and on Azure it works with Virtual Machine Scale Sets. Self-managed clusters require provider-specific configuration.

### How CA Decides to Scale
CA adds a node when it detects a Pod in the Pending state with a reason of `Insufficient cpu`, `Insufficient memory`, or similar. It selects the most appropriate node group based on the resource shape of the pending Pods.

CA removes a node when that node's utilization is below a threshold (default 50%) and all Pods running on it can be successfully moved to other nodes. It respects Pod Disruption Budgets, which means it will not drain a node if doing so would violate the minimum availability guarantees you have set.

### Interaction with HPA
HPA and CA work together naturally. When traffic spikes, HPA first creates new Pod replicas. If the cluster does not have enough capacity to schedule those replicas, they remain Pending. CA then detects these unschedulable Pods and provisions new nodes. Once the nodes are ready, the Pods are scheduled and begin serving traffic. When traffic decreases, HPA scales down the Pods, nodes become underutilized, and CA removes them.

## KEDA (Kubernetes Event-Driven Autoscaling)

KEDA is a CNCF project that extends Kubernetes autoscaling to support event-driven workloads. Rather than scaling based only on CPU and memory, KEDA scales based on the length of a message queue, the lag of a Kafka consumer group, the number of pending HTTP requests, or virtually any other external signal.

KEDA works by acting as a custom metrics server and by introducing a `ScaledObject` custom resource. You define what triggers to watch and what target workload to scale. KEDA then adjusts the HPA on your behalf.

Example use cases:
- Scale a worker Deployment from zero to many replicas when messages arrive in an AWS SQS queue, then scale back to zero when the queue is empty.
- Scale a Kafka consumer Deployment based on consumer group lag, so processing keeps up with producer throughput.
- Scale a batch processing Job based on a Prometheus metric indicating the size of a pending work queue.
- Use a cron trigger to pre-scale services before predictable traffic peaks.

KEDA is especially popular for serverless-style workloads because it can scale all the way to zero, eliminating the cost of idle compute entirely.

## Autoscaling Best Practices

Setting resource requests and limits on every container is not optional when using HPA. The HPA controller calculates CPU utilization as a percentage of the Pod's CPU request. If requests are not set, the metric is undefined and HPA cannot function correctly. Always define `resources.requests.cpu` and `resources.requests.memory` in your container specs.

Combine HPA with the Cluster Autoscaler for a complete solution. HPA handles fast pod-level reactions within seconds to minutes, while CA handles slower node-level provisioning that takes one to five minutes. Together they cover the full range of scaling needs.

Avoid running HPA and VPA in conflicting modes on the same Deployment. This is a common mistake that leads to unstable behavior and unpredictable replica counts.

Set sensible minimum and maximum replica counts. A minimum of zero can be appropriate for batch workers using KEDA, but web-facing services should always maintain at least one or two replicas to avoid cold-start latency during scale-up. A maximum that is too low defeats the purpose of autoscaling during traffic peaks.

Test autoscaling behavior under realistic load before deploying to production. Use load testing tools like k6 or Locust to simulate traffic spikes and verify that HPA reacts within an acceptable time window and that your cluster has the capacity to scale to the required number of nodes.

## Why Autoscaling Matters
Autoscaling is one of the most compelling reasons to adopt Kubernetes. It allows you to build systems that are both cost-efficient at rest and highly available under load — two goals that conflict in a statically provisioned world. A service that is right-sized for average traffic will fail under peak load if it cannot scale. A service over-provisioned for peak traffic will waste money during off-peak hours. Autoscaling bridges this gap, enabling teams to commit to availability SLAs without committing to the infrastructure budget of the worst-case scenario.
