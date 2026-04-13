# Kubernetes Learning Notebook

A structured, hands-on reference guide for learning Kubernetes — from first principles to production-ready deployment patterns. Each module builds on the previous one, combining conceptual explanations with practical commands and real-world examples.

## Table of Contents

| # | Module | Description |
|---|--------|-------------|
| 1 | [Introduction to Kubernetes](MODULE_1_INTRODUCTION.md) | What Kubernetes is, its architecture, core objects, and installation methods |
| 2 | [Core Concepts](MODULE_2_CORE_CONCEPTS.md) | Pods, Deployments, Services, Labels, Namespaces, ConfigMaps, Secrets, ReplicaSets, StatefulSets, DaemonSets |
| 3 | [Deployment Management Strategies](MODULE_3_DEPLOYMENT.md) | Rolling updates, blue-green, canary, scaling, autoscaling, rollbacks, probes, and resource management |
| 4 | [Managing Applications](MODULE_4_MANAGING_APPLICATIONS.md) | kubectl apply/create, lifecycle management, scaling, ConfigMaps/Secrets in practice, health checks, Jobs, CronJobs |
| 5 | [Kubernetes Networking](MODULE_5_NETWORKING.md) | Networking model, Services deep dive, DNS, Ingress, Network Policies, kube-proxy, CNI plugins, service mesh |
| – | [Commands Reference](COMMANDS_REFERENCE.md) | Quick-reference cheat sheet of the most commonly used `kubectl` commands |

## How to Use This Notebook

1. **Follow the modules in order** if you are new to Kubernetes — each module introduces concepts that later ones depend on.
2. **Jump to a specific module** if you already have background knowledge and want to focus on a particular topic.
3. **Consult the Commands Reference** at any time for a quick reminder of the most useful `kubectl` commands without reading through full explanations.
4. All content is written in plain Markdown and renders well on GitHub, in VS Code (with the Markdown preview), or any other Markdown viewer.
