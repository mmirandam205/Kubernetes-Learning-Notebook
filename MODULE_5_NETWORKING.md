# Module 5: Kubernetes Networking

## Overview of the Kubernetes Networking Model
Kubernetes uses a flat networking model where every Pod gets its own unique IP address across the entire cluster. This eliminates the need for NAT between Pods and simplifies service-to-service communication. The model requires that all Pods can communicate with all other Pods without translation, and that nodes can communicate with all Pods. This is implemented by a Container Network Interface (CNI) plugin that you choose when setting up the cluster.

Core networking rules:
- Every Pod has a unique cluster-wide IP address
- Pods on the same node communicate via a virtual bridge
- Pods on different nodes communicate via the CNI overlay or underlay network
- Services provide a stable virtual IP (ClusterIP) in front of a group of Pods

## Services (Deep Dive)
A Service is a stable abstraction that exposes a set of Pods under a consistent DNS name and IP address. Because Pods are ephemeral and their IPs change, Services provide a reliable endpoint. Kubernetes supports several Service types depending on the exposure level needed.

Service types:
- **ClusterIP** (default): Exposes the Service on an internal cluster IP. Only reachable from within the cluster.
- **NodePort**: Exposes the Service on a static port on every node's IP. Accessible from outside the cluster using `<NodeIP>:<NodePort>`.
- **LoadBalancer**: Provisions an external load balancer from a cloud provider (AWS ELB, GCP LB, Azure LB) and routes traffic to the Service.
- **ExternalName**: Maps a Service to an external DNS name (e.g., `my.database.example.com`) by returning a CNAME record.
- **Headless Services**: Set `clusterIP: None` to skip the virtual IP and let DNS return individual Pod IPs directly — useful for StatefulSets and service discovery.

## DNS in Kubernetes
Kubernetes runs CoreDNS as the cluster DNS server by default. Every Service gets a DNS entry automatically, allowing Pods to resolve Services by name rather than IP. This is essential for loosely-coupled microservices that need to discover each other dynamically.

DNS naming conventions:
- Service DNS format: `<service-name>.<namespace>.svc.cluster.local`
- Short form within the same namespace: `<service-name>`
- Pod DNS format: `<pod-ip-dashes>.<namespace>.pod.cluster.local`
- CoreDNS is configured via a ConfigMap in the `kube-system` namespace

## Ingress
An Ingress is a Kubernetes API object that manages external HTTP and HTTPS access to Services inside the cluster. Unlike a LoadBalancer Service (one per Service), a single Ingress can route traffic to many Services based on hostnames and URL paths, reducing cost and complexity.

Key Ingress concepts:
- **Ingress Controller**: The component that fulfills the Ingress rules (e.g., nginx-ingress, Traefik, AWS ALB Ingress Controller). Must be installed separately.
- **Path-based routing**: Route `/api` to one Service and `/web` to another using path rules.
- **Host-based routing**: Route `api.example.com` to one Service and `app.example.com` to another.
- **TLS termination**: Attach a TLS Secret to an Ingress to terminate HTTPS at the controller level.

Example Ingress rule structure:
```yaml
rules:
  - host: app.example.com
    http:
      paths:
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: api-service
              port:
                number: 80
```

## Network Policies
By default, all Pods in a Kubernetes cluster can communicate with each other freely. Network Policies let you restrict that communication using label selectors to define which Pods can send (ingress) or receive (egress) traffic. They are enforced by the CNI plugin, so your chosen plugin must support Network Policies (Calico, Cilium, and Weave do; Flannel does not by default).

Key concepts:
- **podSelector**: Selects the Pods the policy applies to
- **namespaceSelector**: Allows traffic from Pods in specific namespaces
- **ingress rules**: Define which sources may send traffic to the selected Pods
- **egress rules**: Define which destinations the selected Pods may send traffic to
- A Pod with no Network Policy applied has no restrictions — allow-all is the default

## kube-proxy and Traffic Routing
kube-proxy runs on every node and is responsible for implementing the Service abstraction by maintaining network rules. When a Service is created, kube-proxy programs the node's networking layer so that traffic to the Service's ClusterIP is forwarded to one of the healthy backend Pods.

kube-proxy modes:
- **iptables mode** (default): Programs Linux iptables rules for each Service and endpoint. Simple and battle-tested, but can be slow with thousands of Services due to linear rule matching.
- **IPVS mode**: Uses Linux IPVS (IP Virtual Server) for load balancing. Faster and more scalable than iptables, with support for multiple load-balancing algorithms (round-robin, least connections, etc.).
- **userspace mode** (legacy): Traffic passes through a userspace proxy process. Rarely used today due to performance overhead.

## CNI Plugins
The Container Network Interface (CNI) is a standard that defines how networking should be set up for containers. When a Pod is scheduled, the CNI plugin is responsible for creating the Pod's network interface, assigning it an IP address, and configuring routing. Kubernetes does not include a CNI plugin — you must choose and install one.

Popular CNI plugins:
- **Flannel**: Simple overlay network using VXLAN. Easy to set up, but lacks Network Policy support.
- **Calico**: High-performance plugin that supports both overlay and BGP routing, plus full Network Policy enforcement.
- **Weave**: Creates a mesh network between nodes with built-in Network Policy support and encryption.
- **Cilium**: Uses eBPF for high-performance networking, security, and observability. The most feature-rich modern option.

Choosing a CNI plugin depends on your environment, performance requirements, and need for Network Policy support.

## Service Mesh (Introduction)
A service mesh is an infrastructure layer that handles service-to-service communication inside a cluster, providing features like mutual TLS (mTLS), traffic management, retries, circuit breaking, and observability — without changing application code. It works by injecting a sidecar proxy (e.g., Envoy) alongside every Pod.

Popular service mesh options:
- **Istio**: The most feature-rich option, offering fine-grained traffic control, mTLS, and deep observability. Steeper learning curve.
- **Linkerd**: Lightweight and easier to operate than Istio. Focuses on simplicity and performance for Kubernetes-native workloads.
- **Consul Connect**: HashiCorp's service mesh, integrating tightly with Consul for service discovery across Kubernetes and non-Kubernetes environments.

Use a service mesh when you need zero-trust networking, detailed per-request telemetry, or advanced traffic shaping (A/B testing, canary releases) across many microservices.

## Why Networking Matters in Kubernetes
Networking is the backbone that connects all workloads in a Kubernetes cluster. A solid understanding of Services, DNS, Ingress, and Network Policies allows you to design secure, scalable, and maintainable systems. Misconfigured networking is one of the most common sources of production incidents in Kubernetes, making this knowledge essential for anyone operating clusters in real environments.