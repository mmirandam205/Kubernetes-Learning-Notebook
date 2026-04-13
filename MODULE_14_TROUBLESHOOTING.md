# Module 14: Troubleshooting Kubernetes

Troubleshooting is one of the most critical skills for anyone working with Kubernetes in production. When things go wrong — and they will — knowing where to look, what commands to run, and how to interpret the output is what separates a confident Kubernetes engineer from one who is guessing. This module covers the systematic approach to diagnosing and resolving the most common problems across Pods, nodes, networking, storage, and the control plane. It maps directly to the **Troubleshooting domain of the CKA exam**, which carries 30% of the exam weight.

---

## 1. The Troubleshooting Mindset

Before running any command, adopt a structured diagnostic approach. Start at the highest level (the cluster and namespace) and progressively narrow down to the specific failing resource. The four-step framework is:

1. **Observe** — what symptom is visible? Is a Pod crash-looping, a Service unreachable, a node NotReady?
2. **Locate** — which resource is affected? Which namespace, which node?
3. **Describe** — use `kubectl describe` to read events and status conditions.
4. **Logs** — use `kubectl logs` to read application and container output.

Always start broad and get specific. Do not jump straight to restarting Pods without first understanding the cause.

---

## 2. Diagnosing Pod Failures

Pods are the most common source of issues. The first step is always to check the Pod status.

```bash
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -o wide   # shows node placement and IP
```

### Common Pod Status Values and Their Meaning

| Status | Meaning |
|--------|---------|
| `Pending` | Pod is waiting to be scheduled — no suitable node found, or PVC not bound |
| `CrashLoopBackOff` | Container starts, crashes, and Kubernetes keeps restarting it with exponential backoff |
| `ImagePullBackOff` | Kubernetes cannot pull the container image — wrong name, tag, or missing pull secret |
| `ErrImagePull` | First failed attempt to pull the image (precedes ImagePullBackOff) |
| `OOMKilled` | Container exceeded its memory limit and was killed by the kernel |
| `Error` | Container exited with a non-zero exit code |
| `Terminating` | Pod is being deleted but has not yet completed graceful shutdown |
| `ContainerCreating` | Container is being started — may indicate a slow image pull or volume mount issue |
| `Completed` | Container ran to completion (normal for Jobs) |

### Step 1 — Describe the Pod

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Focus on the **Events** section at the bottom. It shows the most recent activity including scheduling decisions, image pulls, probe failures, and OOMKills.

Key fields to check in the output:
- `Status` — current phase
- `Conditions` — Ready, Initialized, ContainersReady, PodScheduled
- `Containers > State` — Running, Waiting (with reason), or Terminated (with exit code)
- `Events` — chronological log of what Kubernetes has done with this Pod

### Step 2 — Read Container Logs

```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -c <container-name>   # for multi-container Pods
kubectl logs <pod-name> -n <namespace> --previous             # logs from the last crashed container
kubectl logs <pod-name> -n <namespace> --tail=100 -f          # stream last 100 lines live
```

Use `--previous` whenever a Pod is in CrashLoopBackOff — it shows the logs of the container run that just failed, not the current (empty) startup attempt.

### Step 3 — Exec Into a Running Container

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl exec -it <pod-name> -n <namespace> -c <container-name> -- /bin/bash
```

Use this to inspect the filesystem, check environment variables, test network connectivity, or run diagnostics from inside the container.

### Diagnosing CrashLoopBackOff

CrashLoopBackOff means the container starts and immediately exits. Common causes:

- **Application error at startup** — wrong configuration, missing environment variable, or misconfigured ConfigMap/Secret. Check logs with `--previous`.
- **Wrong command or entrypoint** — the `command` or `args` in the Pod spec overrides the image's default and may be incorrect.
- **Missing dependency** — the app tries to connect to a database or service that is not available at startup.
- **Permission error** — the process tries to write to a read-only filesystem or bind to a privileged port without the correct security context.
- **OOMKill at startup** — the container needs more memory than its limit allows. Check `kubectl describe` for `OOMKilled` in the last state.

### Diagnosing ImagePullBackOff

```bash
kubectl describe pod <pod-name> | grep -A5 Events
```

Common causes:
- **Typo in image name or tag** — `nginx:lates` instead of `nginx:latest`
- **Image does not exist** — the tag was deleted from the registry
- **Private registry without imagePullSecret** — create a Secret of type `kubernetes.io/dockerconfigjson` and reference it in the Pod spec under `imagePullSecrets`
- **Registry rate limit** — Docker Hub applies pull rate limits to unauthenticated requests

```bash
# Create an image pull secret
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<password> \
  -n <namespace>
```

### Diagnosing OOMKilled

```bash
kubectl describe pod <pod-name> | grep -A3 "Last State"
```

The output will show `Reason: OOMKilled` and `Exit Code: 137`. Solutions:
- Increase the container's `resources.limits.memory`
- Profile the application to find memory leaks
- Use VPA to get a recommended memory limit based on actual usage

### Diagnosing Pending Pods

A Pod stuck in `Pending` is not yet scheduled. Run:

```bash
kubectl describe pod <pod-name> | grep -A10 Events
```

Common reasons:
- **Insufficient CPU or memory** — no node has enough allocatable resources. Check with `kubectl describe nodes` and look at `Allocatable` vs `Requests`.
- **No matching node selector or affinity** — the Pod requires a label that no node has. Check `nodeSelector` and `affinity` in the Pod spec.
- **Taint not tolerated** — the Pod does not have a matching toleration. Check node taints with `kubectl describe node <node-name> | grep Taint`.
- **PVC not bound** — the Pod is waiting for a PersistentVolumeClaim to be bound. Check `kubectl get pvc -n <namespace>`.

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

---

## 3. Diagnosing Service and Networking Issues

When an application is running but cannot be reached, the problem is usually in the Service, DNS, or network policy layer.

### Check the Service

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
```

Key things to verify:
- **Endpoints** — `kubectl get endpoints <service-name> -n <namespace>`. If the Endpoints object is empty (`<none>`), the Service selector does not match any Pod labels.
- **Port mapping** — the Service `targetPort` must match the container's `containerPort`.
- **Protocol** — TCP vs UDP must match.

### Verify Label Selector Matches Pods

```bash
kubectl get pods -n <namespace> --show-labels
kubectl describe svc <service-name> -n <namespace> | grep Selector
```

The Selector on the Service must exactly match the labels on the Pods. A single typo (e.g., `app: nginx` vs `app: Nginx`) means the Service routes to nothing.

### Test Connectivity from Inside the Cluster

```bash
# Run a temporary debug pod
kubectl run debug --image=busybox --rm -it --restart=Never -- /bin/sh

# Inside the debug pod, test:
wget -qO- http://<service-name>.<namespace>.svc.cluster.local
nslookup <service-name>.<namespace>.svc.cluster.local
nc -zv <service-name> <port>
```

### DNS Troubleshooting

DNS is provided by CoreDNS. If DNS resolution fails:

```bash
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl describe configmap coredns -n kube-system
```

Test DNS resolution from a Pod:

```bash
kubectl exec -it <pod-name> -- nslookup kubernetes.default
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

If `/etc/resolv.conf` does not point to the cluster DNS IP (typically `10.96.0.10`), the Pod's DNS policy may be misconfigured. Check `dnsPolicy` in the Pod spec.

### Network Policies Blocking Traffic

If connectivity works without NetworkPolicy but fails after applying one, the policy is likely too restrictive.

```bash
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>
```

To test, temporarily delete the NetworkPolicy and verify connectivity is restored. Then refine the policy rules to allow the required traffic.

### Ingress Not Routing Traffic

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
kubectl get pods -n ingress-nginx   # or whichever namespace the controller runs in
kubectl logs -n ingress-nginx <controller-pod>
```

Common issues:
- Ingress controller is not installed or not running
- `host` field in the Ingress does not match the request's `Host` header
- TLS secret does not exist or is misconfigured
- Backend Service name or port in the Ingress rule is wrong

---

## 4. Diagnosing Node Issues

### Check Node Status

```bash
kubectl get nodes
kubectl describe node <node-name>
```

A node in `NotReady` status means the control plane has lost contact with the kubelet on that node. Look at the **Conditions** section in `describe node` for details — check `MemoryPressure`, `DiskPressure`, `PIDPressure`, and `Ready`.

### SSH Into the Node and Check kubelet

```bash
# On the node itself
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager
```

If kubelet has stopped, restart it:

```bash
systemctl restart kubelet
```

### Check Node Resource Pressure

```bash
kubectl top nodes           # requires metrics-server
kubectl describe node <node-name> | grep -A10 "Allocated resources"
```

If a node is under memory or disk pressure, Kubernetes will evict Pods. Free up resources or add more nodes.

### Cordon and Drain a Node for Maintenance

```bash
# Prevent new Pods from being scheduled on the node
kubectl cordon <node-name>

# Evict existing Pods gracefully
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Re-enable scheduling after maintenance
kubectl uncordon <node-name>
```

---

## 5. Diagnosing Storage Issues

### PVC Stuck in Pending

```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
```

Common causes:
- **No matching PersistentVolume** — no PV exists that satisfies the PVC's storage class, access mode, and size requirements
- **StorageClass does not exist** — the `storageClassName` in the PVC references a class that has not been created
- **Dynamic provisioner not running** — the storage class provisioner Pod is down
- **Access mode mismatch** — the PVC requests `ReadWriteMany` but the storage backend only supports `ReadWriteOnce`

```bash
kubectl get storageclass
kubectl get pv
```

### Pod Cannot Mount Volume

```bash
kubectl describe pod <pod-name> | grep -A10 Events
```

Look for events like `FailedMount`, `FailedAttachVolume`, or `Unable to attach or mount volumes`. Common causes:
- The PVC is bound to a PV on a different node (for `ReadWriteOnce` volumes)
- The volume is still attached to a terminated node
- Filesystem permissions prevent the container from writing to the mount path

---

## 6. Diagnosing Control Plane Issues

For clusters you manage (kubeadm-based), control plane component failures require SSH access to the control plane node.

### Check Control Plane Pods

```bash
kubectl get pods -n kube-system
kubectl describe pod <control-plane-pod> -n kube-system
kubectl logs <control-plane-pod> -n kube-system
```

Control plane components run as static Pods managed by kubelet directly from manifests in `/etc/kubernetes/manifests/`. If a component like `kube-apiserver` is not running, check the manifest file on the node.

### Check etcd Health

```bash
kubectl exec -it etcd-<node> -n kube-system -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

### Check API Server Logs

```bash
kubectl logs kube-apiserver-<node> -n kube-system --tail=50
```

If the API server is completely down and kubectl is not working, check the static pod log directly on the node:

```bash
crictl logs <apiserver-container-id>
```

---

## 7. Useful Diagnostic Commands Quick Reference

### Cluster-Wide Overview

```bash
kubectl get all -A                          # all resources in all namespaces
kubectl get events -A --sort-by='.lastTimestamp'   # all recent events sorted by time
kubectl get componentstatuses               # control plane health (deprecated in 1.19+ but still useful)
```

### Pod Diagnostics

```bash
kubectl describe pod <pod>                  # full status, conditions, events
kubectl logs <pod> --previous               # logs from last crashed container
kubectl exec -it <pod> -- /bin/sh           # interactive shell
kubectl get pod <pod> -o yaml               # full Pod spec and status as YAML
```

### Node Diagnostics

```bash
kubectl top nodes                           # CPU/memory usage per node
kubectl describe node <node>                # full node status, taints, conditions, allocations
kubectl get node <node> -o yaml             # full node spec as YAML
```

### Networking Diagnostics

```bash
kubectl get endpoints <svc>                 # verify service selector matches pods
kubectl run debug --image=busybox --rm -it --restart=Never -- /bin/sh  # ad-hoc debug pod
kubectl port-forward svc/<svc> 8080:80      # forward a service port locally for testing
```

### Resource Usage

```bash
kubectl top pods -n <namespace>             # CPU/memory per pod
kubectl top pods -n <namespace> --containers  # CPU/memory per container
kubectl resource-capacity                   # (requires plugin) node capacity overview
```

---

## 8. Troubleshooting Checklist

Use this checklist when diagnosing any Kubernetes issue:

- [ ] `kubectl get pods -n <namespace>` — Is the Pod running? What is the status?
- [ ] `kubectl describe pod <pod>` — What do the Events say?
- [ ] `kubectl logs <pod> --previous` — What did the container print before it crashed?
- [ ] `kubectl get endpoints <svc>` — Does the Service have any backends?
- [ ] `kubectl get pvc -n <namespace>` — Are PersistentVolumeClaims bound?
- [ ] `kubectl get nodes` — Are all nodes in Ready state?
- [ ] `kubectl top nodes` and `kubectl top pods` — Is any node or Pod under resource pressure?
- [ ] `kubectl get events -n <namespace> --sort-by=.lastTimestamp` — What has happened recently?
- [ ] Check CoreDNS logs if DNS resolution is failing
- [ ] Check NetworkPolicy if traffic is being blocked unexpectedly
- [ ] Check Ingress controller logs if external routing is broken

---

## 9. Why Troubleshooting Matters

Troubleshooting is not just a skill for fixing broken clusters — it is a skill that makes you a better designer. When you have debugged enough CrashLoopBackOff errors, you naturally start writing liveness and readiness probes. When you have chased enough Pending Pods, you start setting accurate resource requests from day one. When you have fixed enough broken Services, you double-check your label selectors before applying manifests. The patterns in this module are also the patterns tested most heavily in the CKA exam, making this module essential preparation for certification.