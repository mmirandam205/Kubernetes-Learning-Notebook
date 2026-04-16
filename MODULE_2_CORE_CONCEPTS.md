# Core Concepts in Kubernetes

## Pods
A Pod is the smallest deployable unit in Kubernetes, which can contain one or more containers. Pods share the same network namespace, and they can communicate with each other through localhost.

**Example YAML:**
```yaml
apiVersion: v1          # API version for core objects like Pods
kind: Pod               # The type of Kubernetes object
metadata:
  name: my-pod          # Unique name for this Pod in the namespace
  labels:
    app: my-app         # Key-value label used to identify/select this Pod
spec:
  containers:
    - name: my-container        # Name of the container inside the Pod
      image: nginx:1.25         # Docker image to run (name:tag)
      ports:
        - containerPort: 80     # Port the container listens on (for documentation/networking)
```
> **Key lines:** `kind: Pod` defines the object type. `spec.containers` lists every container in the Pod. `image` specifies what Docker image to pull. `containerPort` documents the port the app listens on.

---

## Deployments
A Deployment provides declarative updates to Pods and ReplicaSets. It allows you to define the desired state of your application, and Kubernetes will manage the deployment to achieve that state.

**Example YAML:**
```yaml
apiVersion: apps/v1      # API group for workload objects (Deployments, ReplicaSets, etc.)
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3            # Number of identical Pod copies to keep running at all times
  selector:
    matchLabels:
      app: my-app        # Deployment manages Pods that have this label
  template:              # Blueprint used to create each Pod
    metadata:
      labels:
        app: my-app      # Must match selector.matchLabels above
    spec:
      containers:
        - name: my-container
          image: nginx:1.25
          ports:
            - containerPort: 80
```
> **Key lines:** `replicas` controls horizontal scale. `selector.matchLabels` links the Deployment to its Pods. `template` is the Pod spec that gets stamped out for every replica.

---

## Services
A Service in Kubernetes is an abstraction that defines a logical set of Pods and a policy by which to access them. It provides stable network identities and load balancing for the Pods.

**Example YAML:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app          # Routes traffic to all Pods with this label
  ports:
    - protocol: TCP
      port: 80           # Port exposed by the Service (clients connect here)
      targetPort: 80     # Port on the Pod that receives the traffic
  type: ClusterIP        # ClusterIP = internal only; NodePort or LoadBalancer for external access
```
> **Key lines:** `selector` determines which Pods receive traffic. `port` vs `targetPort` lets you decouple the Service port from the container port. `type` controls network visibility.

---

## Labels
Labels are key-value pairs attached to objects such as Pods, Services, and Deployments that can be used to organize and select subsets of objects.

**Example YAML:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: my-app          # Application identifier
    env: production      # Environment tag (dev / staging / production)
    tier: frontend       # Logical tier of the application
spec:
  containers:
    - name: my-container
      image: nginx:1.25
```
> **Key lines:** Labels live under `metadata.labels` as free-form key-value pairs. They are used by `selector` fields in Deployments, Services, and other controllers to target specific Pods.

---

## Selectors
Selectors are the mechanism Kubernetes uses to filter and target objects by their labels. A selector says "give me all objects whose labels match these criteria." There are two types: **equality-based** (uses `=` / `!=`) and **set-based** (uses `in` / `notin` / `exists`).

**Example YAML — equality-based selector in a Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app          # Equality-based: select Pods where label app = my-app
    env: production      # AND label env = production (all conditions must match)
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**Example YAML — set-based selector in a Job:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  selector:
    matchLabels:
      app: my-app                 # Shorthand equality check
    matchExpressions:
      - key: env                  # Set-based: env must be one of these values
        operator: In
        values:
          - production
          - staging
      - key: tier                 # Set-based: tier must NOT be "backend"
        operator: NotIn
        values:
          - backend
  template:
    metadata:
      labels:
        app: my-app
        env: production
        tier: frontend
    spec:
      containers:
        - name: my-container
          image: nginx:1.25
```
> **Key lines:** `selector.matchLabels` is syntactic sugar for equality checks. `selector.matchExpressions` supports richer set-based filtering with `In`, `NotIn`, `Exists`, and `DoesNotExist` operators. All conditions in a selector are ANDed together.

---

## Annotations
Annotations are key-value pairs, just like labels, but they are **not** used for selection or grouping. Instead, they carry arbitrary non-identifying metadata — things like build info, deployment timestamps, tool configuration, or URLs — that tools and humans can read.

**Example YAML:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  annotations:
    kubernetes.io/change-cause: "Bumped nginx from 1.24 to 1.25"   # Tracks rollout reason (used by kubectl rollout history)
    prometheus.io/scrape: "true"                                    # Tells Prometheus to scrape this workload
    prometheus.io/port: "9090"                                      # Port Prometheus should scrape
    owner: "platform-team"                                          # Human-readable ownership info
    docs: "https://wiki.example.com/my-deployment"                  # Link to runbook / documentation
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        co.elastic.logs/enabled: "true"    # Pod-level annotation telling the log agent to collect logs
    spec:
      containers:
        - name: my-container
          image: nginx:1.25
```
> **Key lines:** Annotations live under `metadata.annotations`. Unlike labels, they **cannot** be used in `selector` fields. Values must always be strings (wrap numbers and booleans in quotes). Common uses include rollout history (`kubernetes.io/change-cause`), monitoring scrape config, and CI/CD build metadata.

---

## Namespaces
Namespaces are a way to divide cluster resources between multiple users or teams. They enable the creation of multiple environments within the same cluster, like development, testing, and production.

**Example YAML:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development      # Creates an isolated "development" workspace in the cluster
```
To deploy a Pod into that namespace:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: development  # Assigns this Pod to the "development" namespace
spec:
  containers:
    - name: my-container
      image: nginx:1.25
```
> **Key lines:** `kind: Namespace` creates the isolation boundary. `metadata.namespace` on any object assigns it to that boundary.

---

## ConfigMaps
ConfigMaps are used to store configuration data in key-value pairs, which can be consumed by Pods as environment variables or as configuration files in a volume.

**Example YAML:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_ENV: "production"        # Key-value pair consumed as an env variable
  APP_PORT: "8080"             # Another configuration value
  config.properties: |         # A whole file can be stored as a multi-line value
    log.level=INFO
    max.connections=100
```
Consuming it in a Pod:
```yaml
spec:
  containers:
    - name: my-container
      image: nginx:1.25
      envFrom:
        - configMapRef:
            name: my-config    # Injects ALL keys from the ConfigMap as env variables
```
> **Key lines:** `data` holds the configuration. `envFrom.configMapRef` injects every key as an environment variable; alternatively, use `volumeMounts` to mount it as a file.

---

## Secrets
Secrets are a special type of ConfigMap intended to hold sensitive data such as passwords, OAuth tokens, and SSH keys. Secrets are base64 encoded and can be mounted into Pods as volumes or exposed as environment variables.

**Example YAML:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque              # Generic secret type; use kubernetes.io/tls for TLS certs, etc.
data:
  username: bXl1c2Vy      # base64-encoded value of "myuser"
  password: bXlwYXNz      # base64-encoded value of "mypass"
```
Consuming it in a Pod:
```yaml
spec:
  containers:
    - name: my-container
      image: nginx:1.25
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret   # Name of the Secret object
              key: password     # Specific key inside the Secret to expose
```
> **Key lines:** `type: Opaque` is the default for arbitrary secrets. Values under `data` **must** be base64-encoded. `secretKeyRef` injects a single key as an environment variable without exposing the rest of the Secret.

---

## ReplicaSets
A ReplicaSet ensures that a specified number of Pod replicas are running at any given time. It is commonly used as a part of a Deployment.

**Example YAML:**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3             # Always maintain exactly 3 running Pods
  selector:
    matchLabels:
      app: my-app         # Manages Pods that carry this label
  template:
    metadata:
      labels:
        app: my-app       # New Pods are created with this label
    spec:
      containers:
        - name: my-container
          image: nginx:1.25
```
> **Key lines:** `replicas` is the heart of a ReplicaSet — it self-heals by creating or deleting Pods to match this count. In practice, prefer a **Deployment** over a bare ReplicaSet so you also get rolling updates and rollback support.

---

## StatefulSets
StatefulSets are used to manage the deployment and scaling of a set of Pods, and they offer guarantees about the ordering and uniqueness of these Pods, making them ideal for stateful applications.

**Example YAML:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
spec:
  serviceName: "my-service"   # Headless Service that gives each Pod a stable DNS name
  replicas: 3
  selector:
    matchLabels:
      app: my-stateful-app
  template:
    metadata:
      labels:
        app: my-stateful-app
    spec:
      containers:
        - name: my-container
          image: mysql:8.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql   # Each Pod gets its own persistent volume here
  volumeClaimTemplates:                   # Automatically creates a PVC per Pod replica
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi                  # Each Pod requests 1 GiB of persistent storage
```
> **Key lines:** `serviceName` links to a headless Service so each Pod gets a stable DNS entry like `my-statefulset-0.my-service`. `volumeClaimTemplates` provisions a dedicated PersistentVolumeClaim per replica, ensuring data survives Pod restarts.

---

## DaemonSets
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As Nodes are added to the cluster, Pods are added to them automatically, ensuring consistent behavior across the cluster.

**Example YAML:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane  # Allow running on control-plane nodes too
          effect: NoSchedule
          operator: Exists
      containers:
        - name: log-collector
          image: fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log     # Mounts the host's /var/log to collect node-level logs
      volumes:
        - name: varlog
          hostPath:
            path: /var/log            # Reads directly from the Node filesystem
```
> **Key lines:** A DaemonSet needs **no** `replicas` field — it runs one Pod per Node automatically. `tolerations` let the Pod land on nodes that would otherwise repel it (e.g., control-plane nodes). `hostPath` volumes give the Pod direct access to the Node's filesystem.
