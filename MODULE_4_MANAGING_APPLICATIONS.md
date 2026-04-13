# Module 4: Managing Applications

## Deploying Applications on Kubernetes
Deploying an application on Kubernetes means describing the desired state in a YAML manifest and submitting it to the API Server. Kubernetes then works continuously to ensure the cluster matches that desired state. You can deploy using `kubectl apply -f manifest.yaml` (declarative) or `kubectl create` (imperative). The declarative approach is preferred for production because manifests can be version-controlled and audited.

Key deployment commands:
- `kubectl apply -f deployment.yaml` — create or update resources from a file
- `kubectl create deployment nginx --image=nginx` — imperatively create a deployment
- `kubectl get deployments` — list all deployments in the current namespace
- `kubectl describe deployment <name>` — show detailed information about a deployment

## Managing Application Lifecycle
Once deployed, an application's lifecycle includes updates, pauses, and annotations. Kubernetes makes it easy to update a running application by changing the container image or configuration without downtime. You can pause a rollout to inspect intermediate states and resume it when ready.

Useful lifecycle commands:
- `kubectl set image deployment/<name> <container>=<new-image>` — update container image
- `kubectl rollout pause deployment/<name>` — pause an ongoing rollout
- `kubectl rollout resume deployment/<name>` — resume a paused rollout
- `kubectl annotate deployment/<name> kubernetes.io/change-cause="reason"` — document why a change was made

## Scaling Applications
Kubernetes supports both manual and automatic scaling. Manual scaling lets you instantly adjust the number of Pod replicas. Automatic scaling through the Horizontal Pod Autoscaler (HPA) adjusts replicas based on observed CPU or memory usage. The Vertical Pod Autoscaler (VPA) adjusts resource requests and limits for containers automatically.

Scaling options:
- **Manual scaling**: `kubectl scale deployment/<name> --replicas=5`
- **Horizontal Pod Autoscaler (HPA)**: Scales Pod count based on metrics like CPU utilization
- **Vertical Pod Autoscaler (VPA)**: Adjusts CPU/memory requests and limits per container
- **Cluster Autoscaler**: Adds or removes nodes from the cluster based on pending Pods

## Managing ConfigMaps and Secrets in Practice
ConfigMaps decouple configuration from application code, making apps more portable. Secrets store sensitive information such as passwords and tokens, and are base64-encoded in etcd. Both can be injected into Pods as environment variables or mounted as files in a volume.

Common patterns:
- `kubectl create configmap app-config --from-literal=ENV=production` — create from literal
- `kubectl create configmap app-config --from-file=config.properties` — create from a file
- `kubectl create secret generic db-secret --from-literal=password=s3cr3t` — create a Secret
- Mount a Secret as a volume in a Pod spec using `volumes` and `volumeMounts`
- Inject ConfigMap values as environment variables using `envFrom.configMapRef`

## Rolling Updates and Rollbacks
Rolling updates gradually replace old Pod instances with new ones, ensuring zero downtime. Kubernetes tracks the revision history of each Deployment, allowing you to roll back to any previous version if a problem is detected. The default rolling update strategy keeps a minimum number of Pods available throughout the process.

Key rollout commands:
- `kubectl rollout status deployment/<name>` — watch the progress of a rollout
- `kubectl rollout history deployment/<name>` — view revision history
- `kubectl rollout undo deployment/<name>` — roll back to the previous revision
- `kubectl rollout undo deployment/<name> --to-revision=2` — roll back to a specific revision

## Resource Requests and Limits
Every container in a Pod should declare CPU and memory requests (the minimum guaranteed) and limits (the maximum allowed). The scheduler uses requests to decide which node can host a Pod. If a container exceeds its memory limit, it is OOMKilled (Out of Memory Killed). If it exceeds its CPU limit, it is throttled but not terminated.

Resource management objects:
- **requests**: Minimum CPU/memory the container is guaranteed
- **limits**: Maximum CPU/memory the container may use
- **LimitRange**: Sets default requests/limits for a namespace
- **ResourceQuota**: Caps total resource consumption across an entire namespace

Example resource block in a Pod spec:
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

## Health Checks (Probes)
Kubernetes uses three types of probes to monitor the health of containers. The **liveness probe** determines if a container is still running and restarts it if not. The **readiness probe** determines if a container is ready to receive traffic. The **startup probe** is used for slow-starting containers to prevent premature liveness checks.

Probe types:
- **HTTP GET**: Sends an HTTP request; success if status is 200–399
- **TCP Socket**: Checks if a TCP connection can be established
- **exec**: Runs a command inside the container; success if exit code is 0

Example probe config:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

## Jobs and CronJobs
A **Job** creates one or more Pods and ensures they complete successfully. Jobs are ideal for batch processing, database migrations, or any one-off task. A **CronJob** runs Jobs on a scheduled basis using standard cron syntax, making it suitable for periodic tasks like backups or report generation.

Key characteristics:
- Jobs retry failed Pods until the desired completions count is reached
- `kubectl create job test-job --image=busybox -- echo "hello"` — create a one-off job
- CronJob schedule format: "0 2 * * *" (runs at 2am every day)
- `kubectl get jobs` and `kubectl get cronjobs` — list active jobs and schedules
- Set `restartPolicy: OnFailure` or `restartPolicy: Never` on Job Pods

## Why Proper Application Management Matters
Managing applications well in Kubernetes is the foundation of a reliable production system. By combining declarative deployments, autoscaling, health checks, and proper resource management, teams can ship changes confidently and recover quickly from failures. Understanding these tools ensures your workloads are resilient, observable, and cost-efficient.