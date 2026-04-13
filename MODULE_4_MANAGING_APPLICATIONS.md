# Module 4: Managing Applications

## Deploying Applications on Kubernetes
Deploying applications on Kubernetes involves defining the desired state of your workload through YAML manifests and submitting them to the API server. Kubernetes then reconciles the cluster's actual state with the desired state you declared. There are two primary approaches: **declarative** (recommended) and **imperative**. Understanding the difference between them is key to building reliable and reproducible workflows.

1. **kubectl apply**: The declarative command that creates or updates resources defined in a YAML/JSON file. It compares the current state with the desired state and applies only the differences. This is the preferred approach for production environments where change history matters.
2. **kubectl create**: An imperative command that creates a resource from a file or flags. Unlike `apply`, it will return an error if the resource already exists, making it less suitable for idempotent operations.
3. **Declarative vs imperative approach**: The declarative approach (using `kubectl apply`) treats configuration files as source of truth, allowing GitOps workflows and easy peer review. The imperative approach (using `kubectl run`, `kubectl create`, `kubectl expose`) is faster for quick experiments but harder to audit and repeat reliably.
4. **YAML manifest structure**: A Kubernetes manifest always contains four top-level fields — `apiVersion`, `kind`, `metadata`, and `spec`. The `spec` field varies per resource type and describes the desired configuration in detail.

## Managing Application Lifecycle
Once an application is running, its lifecycle must be actively managed to handle new releases, patches, and configuration changes safely. Kubernetes provides a rich set of commands to update, pause, and annotate deployments without causing downtime. Annotating deployments with meaningful messages is considered best practice because it provides a human-readable audit trail in the rollout history.

1. **Updating images**: Use `kubectl set image deployment/<name> <container>=<new-image>:<tag>` to trigger a rolling update with a new container image. Kubernetes will incrementally replace old pods with new ones, ensuring availability is maintained throughout the update.
2. **Pausing and resuming rollouts**: `kubectl rollout pause deployment/<name>` allows you to temporarily halt a rolling update after making one or more changes, so you can batch multiple changes into a single rollout. Use `kubectl rollout resume deployment/<name>` to continue the paused update.
3. **Annotating deployments**: `kubectl annotate deployment/<name> kubernetes.io/change-cause="<message>"` records a human-readable reason for the update. This annotation is captured in the rollout history and is invaluable when debugging or auditing past changes.

## Scaling Applications
Kubernetes makes it straightforward to scale applications both manually and automatically. Proper scaling ensures that your application can handle varying loads without wasting resources or becoming unavailable. Combining manual scaling for baseline sizing with autoscalers for dynamic load handling is a common production pattern.

1. **Manual scaling**: `kubectl scale deployment/<name> --replicas=<count>` immediately adjusts the number of running pod replicas. This is useful for planned traffic spikes or capacity adjustments that you can anticipate in advance.
2. **Horizontal Pod Autoscaler (HPA)**: The HPA automatically adjusts the number of pod replicas based on observed CPU utilization, memory usage, or custom metrics. It is configured with `kubectl autoscale deployment/<name> --min=<n> --max=<n> --cpu-percent=<threshold>` and continuously queries the Metrics Server to make scaling decisions.
3. **Vertical Pod Autoscaler (VPA)**: The VPA automatically adjusts the CPU and memory **requests and limits** on individual pods rather than changing the replica count. It is useful for workloads with variable resource consumption patterns where horizontal scaling is not appropriate, such as single-threaded or stateful processes.

## Managing ConfigMaps and Secrets in Practice
ConfigMaps and Secrets decouple configuration from application code, making applications more portable and easier to update without rebuilding container images. Both objects can be consumed in multiple ways, giving you flexibility depending on your use case. Always store Secrets in a secure secrets management system (e.g., HashiCorp Vault, AWS Secrets Manager) and sync them into Kubernetes rather than committing them to version control.

1. **Creating ConfigMaps from literals and files**: `kubectl create configmap <name> --from-literal=key=value` creates a ConfigMap inline. `kubectl create configmap <name> --from-file=<path>` reads the file's content as the value, which is useful for full configuration files like `nginx.conf` or `application.properties`.
2. **Mounting Secrets as volumes**: Secrets can be projected into a pod as a volume, where each key becomes a file. This approach is ideal for TLS certificates and SSH keys that applications expect to find on the filesystem, and Kubernetes automatically updates the mounted files when the Secret is updated.
3. **Injecting env vars from ConfigMaps/Secrets**: You can expose ConfigMap or Secret keys as individual environment variables using `envFrom` or `env[].valueFrom` in a pod spec. This is convenient for twelve-factor applications that read their configuration entirely from the environment.

## Rolling Updates and Rollbacks
Rolling updates are Kubernetes' default strategy for updating a deployment without downtime. The process replaces pods incrementally, ensuring a minimum number of pods are always available throughout the transition. If the new version introduces a regression, Kubernetes makes it trivial to roll back to any previous revision.

1. **How rolling updates work step by step**: When a deployment update is triggered, the ReplicaSet controller creates a new ReplicaSet with the updated pod template. It then scales up the new ReplicaSet one pod at a time while simultaneously scaling down the old one, controlled by the `maxSurge` and `maxUnavailable` parameters in the deployment's update strategy.
2. **kubectl rollout status**: `kubectl rollout status deployment/<name>` streams the progress of an ongoing rollout, reporting each pod transition in real time. It exits with a non-zero code if the rollout fails, making it easy to integrate into CI/CD pipelines.
3. **kubectl rollout history**: `kubectl rollout history deployment/<name>` lists all previous revisions of a deployment along with their change-cause annotations. You can inspect a specific revision with `kubectl rollout history deployment/<name> --revision=<n>`.
4. **Rolling back with kubectl rollout undo**: `kubectl rollout undo deployment/<name>` reverts to the previous revision. To target a specific revision, use `kubectl rollout undo deployment/<name> --to-revision=<n>`. Kubernetes keeps a configurable number of old ReplicaSets (controlled by `revisionHistoryLimit`) to support this feature.

## Resource Requests and Limits
Setting resource requests and limits on containers is essential for cluster stability and fair resource sharing. Requests tell the scheduler how much CPU and memory a pod needs, and limits cap how much it can consume at runtime. Without these settings, a single runaway pod can starve other workloads on the same node.

1. **Setting CPU and memory requests/limits**: In the `resources` field of a container spec, `requests` guarantees a minimum allocation and `limits` caps the maximum. CPU is specified in millicores (e.g., `500m` = 0.5 cores) and memory in bytes with suffixes (e.g., `256Mi`, `1Gi`).
2. **LimitRange**: A `LimitRange` object sets default, minimum, and maximum resource values for containers within a namespace. It automatically injects defaults into pods that do not specify their own requests/limits, preventing unguarded workloads from monopolizing node resources.
3. **ResourceQuota**: A `ResourceQuota` places aggregate constraints on the total resource consumption of all objects in a namespace. It can limit total CPU, memory, number of pods, services, and other objects, making it a key tool for multi-tenant cluster governance.
4. **What happens when limits are exceeded**: If a container exceeds its memory limit, it is killed with an **OOMKill** (Out of Memory Kill) signal and restarted by the kubelet according to the pod's restart policy. If a container exceeds its CPU limit, it is **throttled** — its CPU time is artificially restricted — which degrades performance but does not cause a restart.

## Health Checks
Kubernetes uses probes to monitor the health and readiness of containers and take automated corrective actions. Properly configured probes prevent traffic from being sent to pods that are not ready and ensure that unhealthy pods are restarted automatically. Missing or misconfigured probes are a common source of production incidents.

1. **Liveness probes**: A liveness probe tells Kubernetes whether a container is still running correctly. If the probe fails, the kubelet kills the container and restarts it according to the pod's `restartPolicy`. Use liveness probes for detecting deadlocks or corrupted application state that would not otherwise be detected.
2. **Readiness probes**: A readiness probe tells Kubernetes whether a container is ready to receive traffic. If the probe fails, the pod is removed from the endpoints of all matching Services until it passes again. This prevents requests from being sent to a pod that is still initializing or temporarily overloaded.
3. **Startup probes**: A startup probe is used for slow-starting containers that would otherwise be falsely killed by liveness probes before they finish initializing. While the startup probe has not succeeded, both liveness and readiness probes are disabled for that container.
4. **Probe types: HTTP, TCP, exec**: An **HTTP probe** sends a GET request to a specified path and port and considers the container healthy if the response status is between 200 and 399. A **TCP probe** attempts to open a TCP connection to a specified port. An **exec probe** runs an arbitrary command inside the container and considers it healthy if the command exits with code 0.

## Jobs and CronJobs
Not all workloads are long-running services — some are one-off tasks or recurring batch operations. Kubernetes provides `Job` and `CronJob` resources specifically for these use cases, with built-in support for retries, parallelism, and scheduling. Using Jobs instead of plain pods ensures that Kubernetes can automatically retry failed tasks and track completion.

1. **One-off Jobs**: A `Job` creates one or more pods and ensures that a specified number of them successfully complete. If a pod fails, the Job controller automatically retries it up to the number of times specified by `backoffLimit`. Jobs are ideal for database migrations, batch data processing, and any finite task.
2. **Scheduled tasks with CronJobs**: A `CronJob` creates Jobs on a repeating schedule defined in standard cron syntax (e.g., `"0 2 * * *"` for 2 AM daily). Kubernetes creates a new Job object for each scheduled run and retains a configurable number of past successful and failed jobs for inspection.
3. **Restart policies**: Pods in a Job should set `restartPolicy` to `OnFailure` (to restart the container on the same pod) or `Never` (to create a new pod on failure). The `Always` restart policy is reserved for long-running services managed by Deployments and is not permitted for Jobs.

## Why Proper Application Management Matters
Mastering application management in Kubernetes is what separates a functional cluster from a production-ready platform. Without lifecycle management, scaling strategies, and health checks, applications are fragile and difficult to operate reliably at scale.

- **Reliability through automation**: Rolling updates, health probes, and autoscalers work together to keep applications available without constant manual intervention.
- **Operational visibility**: Rollout history, annotations, and monitoring integrations give teams an audit trail and the context needed to diagnose incidents quickly.
- **Resource efficiency**: Requests, limits, LimitRanges, and ResourceQuotas ensure that all workloads on the cluster receive fair access to compute resources without overprovisioning.
- **Separation of concerns**: ConfigMaps and Secrets decouple configuration from code, enabling teams to update application behaviour without touching container images or source code.
- **Resilience by design**: Readiness and liveness probes, combined with Jobs' retry logic, mean that transient failures are handled automatically, reducing on-call burden.
