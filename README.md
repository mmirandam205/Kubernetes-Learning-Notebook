# Kubernetes Learning Notebook

A structured, hands-on reference guide for learning Kubernetes — from first principles to production-ready deployment patterns. Each module builds on the previous one, combining conceptual explanations with real YAML examples and kubectl commands.

## Table of Contents

| # | Module | Description |
|---|--------|-------------|
| 1 | [Introduction to Kubernetes](MODULE_1_INTRODUCTION.md) | What Kubernetes is, its architecture, core objects, and installation methods |
| 2 | [Core Concepts](MODULE_2_CORE_CONCEPTS.md) | Pods, Deployments, Services, Labels, Namespaces, ConfigMaps, Secrets, ReplicaSets, StatefulSets, DaemonSets |
| 3 | [Deployment Management Strategies](MODULE_3_DEPLOYMENT.md) | Rolling updates, blue-green, canary, scaling, autoscaling, rollbacks, probes, and resource management |
| 4 | [Managing Applications](MODULE_4_MANAGING_APPLICATIONS.md) | kubectl apply/create, lifecycle management, scaling, ConfigMaps/Secrets in practice, health checks, Jobs, CronJobs |
| 5 | [Kubernetes Networking](MODULE_5_NETWORKING.md) | Networking model, Services deep dive, DNS, Ingress, Network Policies, kube-proxy, CNI plugins, service mesh |
| 6 | [Storage and Persistent Volumes](MODULE_6_STORAGE.md) | Volumes, PersistentVolumes, PersistentVolumeClaims, StorageClasses, dynamic provisioning, StatefulSet storage |
| 7 | [Security](MODULE_7_SECURITY.md) | Authentication, RBAC, ServiceAccounts, Secrets management, Pod Security, Network Policies, image security |
| 8 | [Observability and Monitoring](MODULE_8_OBSERVABILITY.md) | Logging, metrics, Prometheus, Grafana, distributed tracing, OpenTelemetry |
| 9 | [Autoscaling](MODULE_9_AUTOSCALING.md) | Horizontal Pod Autoscaler (HPA), Vertical Pod Autoscaler (VPA), Cluster Autoscaler, KEDA |
| 10 | [Advanced Scheduling](MODULE_10_ADVANCED_SCHEDULING.md) | Node affinity, taints and tolerations, Pod affinity, topology spread, Priority Classes, resource quotas |
| 11 | [StatefulSets and Operators](MODULE_11_STATEFULSETS_AND_OPERATORS.md) | StatefulSets deep dive, headless Services, Operators, CRDs, Operator SDK, Helm-based operators |
| 12 | [Helm and Package Management](MODULE_12_HELM_AND_PACKAGE_MANAGEMENT.md) | Helm charts, repositories, values, templating, upgrades, rollbacks, creating custom charts |
| 13 | [CI/CD and GitOps](MODULE_13_CICD_AND_GITOPS.md) | GitHub Actions pipelines, ArgoCD, Flux, GitOps principles, image promotion, multi-environment delivery |
| – | [Commands Reference](COMMANDS_REFERENCE.md) | Quick-reference cheat sheet of the most commonly used `kubectl` commands |
| – | [Additional Resources](ADDITIONAL_RESOURCES.md) | Books, courses, certifications, tools, and community links |

## How to Use This Notebook

1. **Follow the modules in order** if you are new to Kubernetes — each module introduces concepts that later ones depend on.
2. **Jump to a specific module** if you already have background knowledge and want to focus on a particular topic.
3. **Consult the Commands Reference** at any time for a quick reminder of the most useful `kubectl` commands without reading through full explanations.
4. All content is written in plain Markdown and renders well on GitHub, in VS Code (with the Markdown preview), or any other Markdown viewer.