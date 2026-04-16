## Network Policies

Network policies are crucial for controlling the traffic flow between pods. They allow the configuration of different communication rules based on labels, namespaces, and ports. Below are two YAML examples demonstrating how to implement network policies effectively.

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
This NetworkPolicy applies to pods in the "beta" namespace (label app: beta-app) and allows ingress on port 8080 from any pod inside the "alpha" namespace (namespace label name: alpha). Traffic from all other namespaces is blocked.

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
