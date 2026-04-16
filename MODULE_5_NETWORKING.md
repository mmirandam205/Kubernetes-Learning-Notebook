## Network Policies

Network policies are crucial for controlling the traffic flow between pods. They allow the configuration of different communication rules based on labels, namespaces, and ports. Below are two YAML examples covering pod-to-pod and namespace-to-namespace communication.

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

---

## CNI Plugins

A **CNI (Container Network Interface) plugin** is responsible for setting up the network for pods — assigning IP addresses, creating virtual network interfaces, and enabling pod-to-pod communication across nodes. Without a CNI plugin, pods cannot communicate with each other.

Kubernetes does **not** ship with a CNI plugin built-in. You must install one. Below are three of the most popular choices, each with a practical configuration example.

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
      "Network": "10.244.0.0/16",   # The overall pod CIDR for the cluster
      "Backend": {
        "Type": "vxlan"             # Encapsulation method: vxlan is the default overlay
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