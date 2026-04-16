# Module 5: Kubernetes Networking

## Overview of Kubernetes Networking Model
Kubernetes networking is built on a **flat networking model** where every pod in the cluster receives its own unique IP address. This eliminates the need for NAT between pods and simplifies service discovery and communication across the cluster.

1. **Flat networking model**: All pods in the cluster share a single flat address space. No NAT is required between pods, meaning the source IP seen by a receiving pod is always the actual IP of the sending pod.
2. **Every pod gets its own IP**: Each pod is assigned an IP from the cluster's pod CIDR range when it is scheduled. Containers within the same pod share that IP and communicate with each other over localhost.
3. **Pod-to-pod, pod-to-service, external traffic**: Pod-to-pod communication is handled directly via the CNI plugin's overlay or underlay network. Pod-to-service traffic is managed by `kube-proxy` using iptables or IPVS rules. External traffic enters through NodePort, LoadBalancer services, or an Ingress controller.

## Services (Deep Dive)
A Kubernetes Service provides a stable virtual IP (ClusterIP) and DNS name that abstract away the ephemeral nature of pod IPs. Because pods are constantly created and destroyed, applications should never reference pod IPs directly — they should always communicate through a Service.

---

### Types of Services

#### 1. ClusterIP
The default Service type. It exposes the service on an internal IP address that is only reachable from within the cluster. It is the correct choice for internal microservice-to-microservice communication.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
spec:
  type: ClusterIP          # Default — only reachable inside the cluster
  selector:
    app: backend           # Routes traffic to pods with this label
  ports:
    - name: http
      port: 80             # Port the Service listens on (cluster-internal)
      targetPort: 8080     # Port the container is actually listening on
      protocol: TCP
```

> **Key points**:
> - A virtual IP (ClusterIP) is automatically assigned from the cluster's service CIDR range.
> - Accessible only from within the cluster — not from outside.
> - The `selector` must match the labels on the target pods.

> **When to use**: Any internal service that does not need to be reached from outside the cluster (e.g., a database, a cache, a backend API consumed only by other pods).

---

#### 2. NodePort
Exposes the service on a static port (in the range 30000–32767) on **every node's IP address**. Any external traffic sent to `<NodeIP>:<NodePort>` is forwarded to the Service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: production
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      port: 80             # Port the Service listens on (cluster-internal)
      targetPort: 3000     # Port the container is listening on
      nodePort: 31000      # Static port opened on every node (30000–32767)
      protocol: TCP
```

> **Key points**:
> - If `nodePort` is omitted, Kubernetes automatically assigns a free port in the 30000–32767 range.
> - Every node in the cluster opens `nodePort`, even nodes not running the pod.
> - Traffic to `<NodeIP>:31000` is load-balanced across all healthy pods matched by the selector.

> **When to use**: Development or testing environments, or when you need simple external access without a cloud load balancer. Not recommended for production internet-facing traffic.

---

#### 3. LoadBalancer
Provisions an external cloud load balancer (e.g., AWS ELB, GCP Cloud Load Balancing) and maps it to the Service. This is the standard way to expose internet-facing services in cloud environments.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-api-service
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"   # AWS: use Network Load Balancer
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - name: https
      port: 443            # Port exposed on the external load balancer
      targetPort: 8443     # Port the container is listening on
      protocol: TCP
  loadBalancerSourceRanges:
    - 0.0.0.0/0            # Allow all IPs; restrict this in production for security
```

> **Key points**:
> - Automatically creates a NodePort and ClusterIP underneath — LoadBalancer is a superset.
> - The cloud provider's load balancer IP/hostname is assigned and visible under `.status.loadBalancer.ingress`.
> - `loadBalancerSourceRanges` restricts which client IPs can reach the load balancer — always lock this down in production.
> - Annotations are provider-specific (AWS, GCP, Azure each have their own).

> **When to use**: Exposing a single service directly to the internet in a cloud environment. For multiple services, prefer Ingress + ClusterIP to avoid provisioning one load balancer per service.

---

#### 4. ExternalName
Maps a Service to an external DNS name rather than to a set of pod endpoints. This is useful for integrating external services (e.g., a managed database) into the cluster's DNS without hardcoding external hostnames in application config.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: managed-db
  namespace: production
spec:
  type: ExternalName
  externalName: my-database.abc123.us-east-1.rds.amazonaws.com   # External DNS hostname
```

> **Key points**:
> - No `selector` is used — no pods are targeted.
> - No ClusterIP is allocated; DNS resolution returns a CNAME pointing to `externalName`.
> - Pods in the cluster resolve `managed-db.production.svc.cluster.local` and get redirected to the external hostname transparently.
> - TLS validation is the client's responsibility — the Service itself does not terminate or inspect traffic.

> **When to use**: Abstracting external managed services (RDS, ElastiCache, a third-party API) so that your application code only ever references an in-cluster DNS name, making it easy to swap the external endpoint without redeploying the application.

---

#### 5. Headless Service
Created by setting `clusterIP: None`, a headless service does not allocate a virtual IP. Instead, DNS queries for the service return the individual pod IPs directly. This is essential for stateful applications where clients need to connect to a specific pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
  namespace: production
  labels:
    app: cassandra
spec:
  type: ClusterIP
  clusterIP: None          # This is what makes it headless — no VIP is allocated
  selector:
    app: cassandra
  ports:
    - name: cql
      port: 9042
      targetPort: 9042
      protocol: TCP
```

> **Key points**:
> - DNS lookup for `cassandra.production.svc.cluster.local` returns **all pod IPs** (A records), not a single virtual IP.
> - When used with a `StatefulSet`, each pod gets a stable, predictable DNS name: `<pod-name>.<service-name>.<namespace>.svc.cluster.local` (e.g., `cassandra-0.cassandra.production.svc.cluster.local`).
> - Clients are responsible for their own load balancing or leader election (e.g., the application picks which pod IP to connect to).
> - `kube-proxy` does **not** process headless services — traffic goes directly to the pod.

> **When to use**: StatefulSets (databases, Kafka, Zookeeper, Cassandra) where each pod is uniquely addressable and stable identity matters, or when you need client-side load balancing rather than kube-proxy's random selection.

---

### Service Type Summary

| Type | Accessible From | VIP Allocated | Use Case |
|---|---|---|---|
| ClusterIP | Inside cluster only | Yes | Internal microservice communication |
| NodePort | Outside cluster via node IP | Yes | Dev/test external access |
| LoadBalancer | Outside cluster via cloud LB | Yes | Production internet-facing services |
| ExternalName | Inside cluster (CNAME redirect) | No | Abstracting external services |
| Headless | Inside cluster (direct pod IPs) | No | StatefulSets, client-side LB |

## DNS in Kubernetes
Kubernetes ships with a cluster-internal DNS server that automatically assigns DNS names to Services and Pods, enabling service discovery by name rather than IP address. Without cluster DNS, every application would need to track pod IPs manually — an impossible task in a dynamic cluster.

1. **CoreDNS**: CoreDNS is the default DNS solution for Kubernetes clusters since version 1.11. It runs as a deployment in the `kube-system` namespace, and `kubelet` configures each pod to use it as its DNS resolver by injecting the CoreDNS ClusterIP into `/etc/resolv.conf` at pod startup.
2. **Service DNS format**: Every Service gets a DNS entry in the format `<service-name>.<namespace>.svc.cluster.local`. Pods in the same namespace can resolve the service by its short name alone (e.g., `backend-service`), while pods in other namespaces must use the full qualified name.
3. **Pod DNS**: Pods can also be given DNS records of the form `<pod-ip-dashes>.<namespace>.pod.cluster.local`. When using headless services with StatefulSets, each pod gets a stable, predictable DNS name tied to its ordinal index.

## Ingress
While Services handle cluster-internal and basic external traffic routing, Ingress provides HTTP and HTTPS routing from outside the cluster to Services within it. Ingress consolidates multiple routing rules into a single resource and is managed by an Ingress controller running inside the cluster.

### What is an Ingress?
An Ingress is a Kubernetes API object that defines rules for routing external HTTP/HTTPS traffic to cluster services based on the request hostname and URL path. By itself, an Ingress object does nothing — it requires an **Ingress controller** to read the rules and configure a reverse proxy accordingly.

### Ingress Controllers
An Ingress controller is a reverse proxy (e.g., **nginx-ingress**, **Traefik**, **HAProxy**, **AWS ALB Ingress Controller**) deployed as a pod in the cluster that watches for Ingress resources and dynamically configures itself to route traffic as defined.

![Kubernetes Ingress Traffic Flow](assets/images/kubernetes-ingress-traffic-flow.png)

---

### Types of Ingress

#### 1. Single-Domain Ingress
Routes all traffic for a single hostname to one backend service. This is the simplest Ingress pattern — one domain, one service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: single-domain-ingress
  namespace: production
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

> **When to use**: A single frontend or API that owns an entire domain.

---

#### 2. Multi-Domain Ingress (Virtual Hosting)
Routes traffic to different backend services based on the `host` header. This allows a single Ingress controller to serve multiple domains, each backed by a different service — similar to virtual hosting in traditional web servers.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-ingress
  namespace: production
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 3000
```

> **When to use**: Multiple distinct applications or subdomains that each own a separate service but share a single Ingress controller.

---

#### 3. Path-Based Ingress
Routes traffic to different services based on the URL path, all under the same hostname. This is the most common pattern for splitting a monolith or grouping microservices under one domain.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  namespace: production
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /metrics
            pathType: Exact
            backend:
              service:
                name: metrics-service
                port:
                  number: 9090
```

> **`pathType` values**:
> - `Prefix` — matches any path that begins with the specified string (e.g., `/api` matches `/api`, `/api/users`, `/api/v2`).
> - `Exact` — matches only the exact path string, no trailing slash or subpaths.
> - `ImplementationSpecific` — matching is left to the Ingress controller.

> **When to use**: A single domain that fronts multiple services differentiated by URL structure (e.g., `/`, `/api`, `/auth`, `/admin`).

---

#### 4. Multi-Service Ingress
Combines both multi-domain and path-based routing in a single Ingress resource. This is the most expressive pattern and is common in production environments with many services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
  namespace: production
spec:
  rules:
    - host: store.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: storefront-service
                port:
                  number: 80
          - path: /cart
            pathType: Prefix
            backend:
              service:
                name: cart-service
                port:
                  number: 8080
          - path: /checkout
            pathType: Prefix
            backend:
              service:
                name: checkout-service
                port:
                  number: 8081
    - host: blog.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog-service
                port:
                  number: 80
```

> **When to use**: Large applications where multiple teams own different subdomains or URL segments, all routed through one Ingress.

---

#### 5. TLS Ingress
Enables HTTPS by referencing a Kubernetes `Secret` that holds a TLS certificate and private key. The Ingress controller terminates the TLS connection and forwards plain HTTP to the backend services.

**Step 1 — Create the TLS Secret:**
```bash
kubectl create secret tls app-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n production
```

**Step 2 — Reference the Secret in the Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"   # Force HTTP → HTTPS redirect
spec:
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: app-tls-secret     # Must exist in the same namespace
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

> **Key points**:
> - The `tls[].hosts` list must match the `rules[].host` values exactly.
> - The Secret must live in the **same namespace** as the Ingress.
> - For automatic certificate management, use **cert-manager** with Let's Encrypt instead of manually managed secrets.

> **When to use**: Any production workload that serves traffic over the internet. TLS should be the default, not an afterthought.

---

#### 6. Default Backend Ingress
A default backend catches all requests that do not match any rule in the Ingress (unmatched hostnames or paths). It is typically used to serve a custom 404 page or to redirect stray traffic to a known endpoint.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
  namespace: production
spec:
  defaultBackend:                    # Catches all unmatched traffic
    service:
      name: default-404-service
      port:
        number: 80
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

> **When to use**: Always recommended in production. Without a default backend, unmatched requests return an unhelpful controller-level error. A dedicated default backend lets you return a branded error page.

---

### Ingress Type Summary

| Type | Routing Criterion | Typical Use Case |
|---|---|---|
| Single-Domain | Hostname only | One app, one domain |
| Multi-Domain | Multiple hostnames | Multiple apps sharing one controller |
| Path-Based | URL path under one hostname | Microservices split by URL |
| Multi-Service | Hostname + path combined | Large multi-team applications |
| TLS | Any of the above + HTTPS | Any internet-facing production service |
| Default Backend | Catch-all for unmatched requests | Custom 404 handling, traffic auditing |

## Network Policies
Network policies are crucial for controlling the traffic flow between pods. They allow the configuration of different communication rules based on labels, namespaces, and ports. Below are two YAML examples covering pod-to-pod and namespace-to-namespace communication.

1. **Default allow-all behavior**: Without any Network Policies, all pods in all namespaces can communicate with each other on any port. This is convenient for development but is a significant security risk in production.
2. **Defining ingress/egress rules**: A Network Policy selects a set of pods (using `podSelector`) and then defines `ingress` (incoming) and `egress` (outgoing) rules. Each rule specifies allowed sources or destinations by pod label, namespace label, or IP block.
3. **Namespace and pod selectors**: `namespaceSelector` allows traffic from all pods in matching namespaces, while `podSelector` restricts to pods with specific labels within those namespaces. Combining both gives fine-grained control.
4. **Use cases: isolating environments**: A common pattern is to place production, staging, and development workloads in separate namespaces and use Network Policies to prevent cross-environment communication.

### Example 1 — Pod-to-pod policy:
This NetworkPolicy applies to the "web" pod (label app: web) which only allows ingress on port 80 from pods labeled with app: api in the same namespace. All other sources are blocked.
#### Pod Definitions:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
  namespace: production
spec:
  containers:
  - name: web
    image: nginx
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
  labels:
    app: api
  namespace: production
spec:
  containers:
  - name: api
    image: myapi:latest
```
#### NetworkPolicy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 80
  policyTypes:
  - Ingress
```

> **Key Lines Callout:**
> - `podSelector`: Targets pods with specific labels.
> - `ingress`: Defines rules for incoming traffic.
> - `implicit deny`: By default, any traffic not explicitly allowed is denied.

### Example 2 — Namespace-to-namespace policy:
This NetworkPolicy applies to pods in the "beta" namespace (label app: beta-app) and allows ingress on port 8080 from any pod inside the "alpha" namespace (namespace label name: alpha). Traffic from any other namespace is blocked.
#### Pod Definitions:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: beta-pod
  labels:
    app: beta-app
  namespace: beta
spec:
  containers:
  - name: beta-app
    image: mybetaapp:latest
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: alpha-pod
  labels:
    app: alpha-app
  namespace: alpha
spec:
  containers:
  - name: alpha-app
    image: myalphaapp:latest
```
#### NetworkPolicy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-beta-from-alpha
  namespace: beta
spec:
  podSelector:
    matchLabels:
      app: beta-app
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: alpha
    ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress
```

> **Key Lines Callout:**
> - `podSelector`: Targets pods with specific labels.
> - `namespaceSelector`: Targets namespaces based on labels.
> - `ingress`: Defines rules for incoming traffic.
> - `implicit deny`: By default, any traffic not explicitly allowed is denied.

## kube-proxy and iptables/IPVS
`kube-proxy` is a component that runs on every node in the cluster and is responsible for implementing the Service abstraction at the network level. It watches the Kubernetes API for Service and Endpoint changes and programs the host's network rules accordingly.

1. **Role of kube-proxy**: kube-proxy maintains network rules that allow network communication to pods from network sessions inside or outside of the cluster. When a packet destined for a ClusterIP arrives at a node, kube-proxy's rules intercept it and redirect it to one of the healthy backend pods.
2. **iptables mode**: In iptables mode (the default), kube-proxy programs Linux `iptables` rules to perform DNAT (Destination NAT) on packets bound for a ClusterIP. It randomly selects a backend pod using statistic rules. Performance degrades as the number of services grows because iptables rules are evaluated linearly.
3. **IPVS mode**: In IPVS (IP Virtual Server) mode, kube-proxy uses the Linux kernel's built-in Layer 4 load balancer, which stores rules in a hash table for O(1) lookup instead of linear iptables traversal. IPVS also supports multiple load-balancing algorithms (round-robin, least connections, source hashing) and is recommended for large clusters with thousands of services.

## CNI Plugins

A **CNI (Container Network Interface) plugin** is responsible for setting up the network for pods — assigning IP addresses, creating virtual network interfaces, and enabling pod-to-pod communication across nodes. Without a CNI plugin, pods cannot communicate with each other.

Kubernetes does **not** ship with a CNI plugin built-in. You must install one. Below are three of the most popular choices, each with a practical configuration example.

1. **What is CNI?**: The Container Network Interface (CNI) is a specification and a set of libraries for writing plugins to configure network interfaces in Linux containers. When a pod is created, the kubelet calls the CNI plugin to set up the pod's network namespace.
2. **Flannel**: A simple and easy-to-configure overlay network that uses VXLAN encapsulation to tunnel pod traffic between nodes. Flannel is a good choice for simple clusters where ease of setup is more important than advanced features.
3. **Calico**: A high-performance CNI plugin that supports both overlay (VXLAN/IP-in-IP) and non-overlay (BGP routing) modes. Calico has native support for Kubernetes Network Policies as well as its own richer policy model.
4. **Weave Net**: An overlay network that creates a virtual network connecting Docker containers across multiple hosts. Weave supports Network Policies and encrypts traffic between nodes, making it a good choice when security is a priority.
5. **Cilium**: A modern CNI plugin that uses **eBPF** (extended Berkeley Packet Filter) technology for networking, security, and observability. Cilium can enforce network policies at the syscall level, providing stronger security guarantees than iptables-based approaches.

---

### Plugin 1 — Flannel
**What it is:** The simplest CNI plugin. It creates a flat overlay network using VXLAN tunnels so that every pod gets a unique IP and can reach any other pod directly.
**Best for:** Learning environments, small clusters, or when you just need something that works without complexity.
#### How to install:
Apply the official Flannel manifest — it deploys a DaemonSet that runs the Flannel agent on every node.
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
#### What gets created (simplified view of the DaemonSet):
```yaml
# This is created automatically by the manifest above.
# Shown here so you understand what Flannel deploys.
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        app: flannel
    spec:
      containers:
      - name: kube-flannel
        image: flannel/flannel:v0.24.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq        # Enables IP masquerading for outbound traffic
        - --kube-subnet-mgr  # Uses Kubernetes API to manage subnets
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```
#### Flannel ConfigMap (network settings):
Flannel stores its network config in a ConfigMap. When you apply the manifest, this is created automatically with these defaults:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```
> **Key Lines Callout:**
> - `Network: 10.244.0.0/16`: The IP range shared across all nodes. Each node gets a `/24` slice (e.g., `10.244.1.0/24`).
> - `Backend Type: vxlan`: Flannel wraps pod traffic inside UDP packets to cross node boundaries.
> - `DaemonSet`: Ensures the Flannel agent runs on **every** node automatically.

---

### Plugin 2 — Calico
**What it is:** A production-grade CNI plugin that provides networking **and** built-in network policy enforcement. Unlike Flannel, Calico can route traffic natively (no overlay needed) using BGP, making it faster.
**Best for:** Production clusters where you need both networking and fine-grained security policies.
#### How to install:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```
#### Key part of the Calico DaemonSet config (auto-generated by the manifest):
```yaml
# calico-node runs on every node and handles routing + policy enforcement
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      containers:
      - name: calico-node
        image: calico/node:v3.27.0
        env:
        - name: DATASTORE_TYPE
          value: "kubernetes"           # Uses Kubernetes API as the data store
        - name: CALICO_IPV4POOL_CIDR
          value: "192.168.0.0/16"       # Pod IP range managed by Calico
        - name: CALICO_IPV4POOL_IPIP
          value: "Off"                  # Disables IP-in-IP overlay (uses native routing)
        - name: FELIX_LOGSEVERITYSCREEN
          value: "info"
```
#### Calico also supports its own policy resource (beyond standard NetworkPolicy):
```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-all-ingress-by-default
spec:
  selector: all()          # Applies to ALL pods in the cluster
  types:
  - Ingress
  ingress: []              # Empty list = deny everything not explicitly allowed
```
> **Key Lines Callout:**
> - `CALICO_IPV4POOL_CIDR`: Defines the pod IP range; must match `--pod-network-cidr` set during `kubeadm init`.
> - `CALICO_IPV4POOL_IPIP: "Off"`: Native BGP routing (no tunnel overhead) — faster than Flannel's VXLAN.
> - `GlobalNetworkPolicy`: Calico-specific resource that can enforce rules cluster-wide, not just per-namespace.
> - `selector: all()`: Targets every pod — useful for setting a secure default-deny baseline.

---

### Plugin 3 — Weave Net
**What it is:** A CNI plugin that creates a mesh network between nodes using encrypted tunnels. Each node connects directly to every other node (full mesh), and traffic is encrypted in transit with no extra configuration.
**Best for:** Clusters where encrypted pod-to-pod traffic is required out of the box, without configuring TLS manually.
#### How to install:
```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
#### Key part of the Weave DaemonSet (auto-generated by the manifest):
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: weave-net
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: weave-net
  template:
    metadata:
      labels:
        name: weave-net
    spec:
      containers:
      - name: weave
        image: weaveworks/weave-kube:2.8.1
        env:
        - name: IPALLOC_RANGE
          value: "10.32.0.0/12"    # Pod IP range assigned by Weave
        - name: CHECKPOINT_DISABLE
          value: "1"               # Disables version-check phone-home
      - name: weave-npc            # Second container: the network policy controller
        image: weaveworks/weave-npc:2.8.1
```
> **Key Lines Callout:**
> - `IPALLOC_RANGE: 10.32.0.0/12`: Weave manages this CIDR and carves out subnets per node automatically.
> - `weave-npc` (sidecar container): Weave ships its own NetworkPolicy controller in the same DaemonSet pod — no separate install needed.
> - **Encryption**: Weave encrypts traffic between nodes by default using NaCl (sleeve mode), unlike Flannel and Calico which require extra steps for encryption.

---

### CNI Plugin Comparison
| Feature | Flannel | Calico | Weave Net |
|---|---|---|---|
| **Overlay / Routing** | VXLAN overlay | Native BGP (no overlay) | Mesh overlay |
| **Network Policy support** | ❌ No | ✅ Yes (+ GlobalNetworkPolicy) | ✅ Yes (via weave-npc) |
| **Encrypted traffic** | ❌ No | ❌ No (optional with WireGuard) | ✅ Yes (default) |
| **Complexity** | Low | Medium | Low–Medium |
| **Best for** | Learning / Dev | Production / Security | Encrypted mesh |

## Service Mesh (Introduction)
As microservice architectures grow in complexity, managing service-to-service communication — including retries, timeouts, circuit breaking, mutual TLS, and observability — at the application layer becomes unsustainable. A service mesh offloads these concerns to a dedicated infrastructure layer.

1. **What is a service mesh?**: A service mesh is an infrastructure layer that controls how services communicate with each other. It consists of a **data plane** (lightweight proxy sidecars, typically Envoy, injected into every pod) and a **control plane** (which configures the proxies and provides a management API).
2. **Istio**: Istio is the most widely adopted service mesh for Kubernetes. It uses Envoy as its sidecar proxy and provides a comprehensive feature set including automatic mTLS between services, traffic management (canary deployments, A/B testing, traffic mirroring), and deep observability via metrics, logs, and traces.
3. **Linkerd**: Linkerd is a lightweight, CNCF-graduated service mesh that prioritizes simplicity and performance. It uses its own purpose-built proxy (written in Rust) instead of Envoy, resulting in a smaller memory footprint and lower latency overhead compared to Istio.
4. **When to use a service mesh**: A service mesh is most valuable when you have many services communicating with each other and you need consistent observability, security (mTLS), and traffic management without modifying application code. For simple clusters with few services, a service mesh adds operational complexity that may not be justified.

## Why Networking Matters in Kubernetes
Networking is the connective tissue of any distributed system, and Kubernetes networking is no exception. A misconfigured network policy, an incorrect Service type, or a poorly chosen CNI plugin can lead to security vulnerabilities, service outages, or performance bottlenecks.

- **Security**: Network Policies and mTLS (via a service mesh) are the primary mechanisms for enforcing zero-trust security principles within the cluster, limiting the blast radius of a compromised pod.
- **Reliability**: Services with proper readiness probes and well-configured load balancing ensure that traffic is only ever sent to healthy pods, eliminating request errors caused by pod churn.
- **Scalability**: Choosing the right Service type (ClusterIP vs. LoadBalancer) and CNI plugin ensures that the networking stack scales with your workload without becoming a bottleneck.
- **Observability**: DNS-based service discovery, combined with distributed tracing tools like Jaeger or Zipkin (often deployed as part of a service mesh), gives operators end-to-end visibility into request flows across services.
- **Operability**: Ingress consolidates external traffic routing into a single, auditable resource, making it easier to manage TLS certificates, routing rules, and rate limiting in one place.