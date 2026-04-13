# Kubernetes Command Reference Guide

This document serves as a comprehensive reference for commonly used `kubectl` commands in Kubernetes. It covers commands related to cluster information, namespaces, pods, deployments, services, configmaps, secrets, and other useful commands.

## Cluster Info

- Get cluster info:
  ```bash
  kubectl cluster-info
  ```

- Get nodes:
  ```bash
  kubectl get nodes
  ```

- Get node details:
  ```bash
  kubectl describe node <node-name>
  ```

## Namespaces

- List all namespaces:
  ```bash
  kubectl get namespaces
  ```

- Create a new namespace:
  ```bash
  kubectl create namespace <namespace-name>
  ```

- Delete a namespace:
  ```bash
  kubectl delete namespace <namespace-name>
  ```

## Pods

- List all pods in the current namespace:
  ```bash
  kubectl get pods
  ```

- Get detailed info about a pod:
  ```bash
  kubectl describe pod <pod-name>
  ```

- Create a pod:
  ```bash
  kubectl run <pod-name> --image=<image-name>
  ```

- Delete a pod:
  ```bash
  kubectl delete pod <pod-name>
  ```

## Deployments

- Create a deployment:
  ```bash
  kubectl create deployment <deployment-name> --image=<image-name>
  ```

- List all deployments:
  ```bash
  kubectl get deployments
  ```

- Update a deployment:
  ```bash
  kubectl set image deployment/<deployment-name> <container-name>=<new-image-name>
  ```

- Delete a deployment:
  ```bash
  kubectl delete deployment <deployment-name>
  ```

## Services

- Create a service:
  ```bash
  kubectl expose deployment <deployment-name> --type=<service-type> --name=<service-name>
  ```

- List all services:
  ```bash
  kubectl get services
  ```

- Get service details:
  ```bash
  kubectl describe service <service-name>
  ```

- Delete a service:
  ```bash
  kubectl delete service <service-name>
  ```

## ConfigMaps

- Create a ConfigMap from a file:
  ```bash
  kubectl create configmap <configmap-name> --from-file=<path>
  ```

- List ConfigMaps:
  ```bash
  kubectl get configmaps
  ```

- Delete a ConfigMap:
  ```bash
  kubectl delete configmap <configmap-name>
  ```

## Secrets

- Create a secret from literal value:
  ```bash
  kubectl create secret generic <secret-name> --from-literal=<key>=<value>
  ```

- List secrets:
  ```bash
  kubectl get secrets
  ```

- Delete a secret:
  ```bash
  kubectl delete secret <secret-name>
  ```

## Miscellaneous Commands

- Get logs of a pod:
  ```bash
  kubectl logs <pod-name>
  ```

- Execute command in a pod:
  ```bash
  kubectl exec -it <pod-name> -- <command>
  ```

- Scale a deployment:
  ```bash
  kubectl scale deployment <deployment-name> --replicas=<number>
  ```

- Port forwarding to a pod:
  ```bash
  kubectl port-forward <pod-name> <local-port>:<pod-port>
  ```

- Get resource usage:
  ```bash
  kubectl top pod
  ```

This is a concise reference guide for `kubectl`. For more detailed information, consult the official Kubernetes documentation.

## Autoscaling

- Create an HPA for a Deployment:
  ```bash
  kubectl autoscale deployment <name> --cpu-percent=50 --min=2 --max=10
  ```

- List all HPAs:
  ```bash
  kubectl get hpa
  ```

- Describe an HPA:
  ```bash
  kubectl describe hpa <name>
  ```

- Show resource usage per Pod:
  ```bash
  kubectl top pods
  ```

- Show resource usage per node:
  ```bash
  kubectl top nodes
  ```

## StatefulSets

- List all StatefulSets:
  ```bash
  kubectl get statefulsets
  ```

- Describe a StatefulSet:
  ```bash
  kubectl describe statefulset <name>
  ```

- Scale a StatefulSet:
  ```bash
  kubectl scale statefulset <name> --replicas=<n>
  ```

- Delete a StatefulSet while keeping its Pods:
  ```bash
  kubectl delete statefulset <name> --cascade=orphan
  ```

## DaemonSets, Jobs, and CronJobs

- List all DaemonSets:
  ```bash
  kubectl get daemonsets
  ```

- List all Jobs:
  ```bash
  kubectl get jobs
  ```

- List all CronJobs:
  ```bash
  kubectl get cronjobs
  ```

- Create a one-off Job:
  ```bash
  kubectl create job <name> --image=<image>
  ```

- Create a CronJob:
  ```bash
  kubectl create cronjob <name> --image=<image> --schedule="*/5 * * * *"
  ```

## Scheduling

- Add a taint to a node:
  ```bash
  kubectl taint nodes <node-name> key=value:NoSchedule
  ```

- Remove a taint from a node:
  ```bash
  kubectl taint nodes <node-name> key=value:NoSchedule-
  ```

- Label a node:
  ```bash
  kubectl label nodes <node-name> <key>=<value>
  ```

- List all PriorityClasses:
  ```bash
  kubectl get priorityclasses
  ```

## Helm

- Add a Helm repository:
  ```bash
  helm repo add <name> <url>
  ```

- Update repository cache:
  ```bash
  helm repo update
  ```

- Search for charts:
  ```bash
  helm search repo <keyword>
  ```

- Install a chart as a named release:
  ```bash
  helm install <release-name> <chart>
  ```

- Upgrade a release:
  ```bash
  helm upgrade <release-name> <chart>
  ```

- Roll back a release to a previous revision:
  ```bash
  helm rollback <release-name> <revision>
  ```

- Uninstall a release:
  ```bash
  helm uninstall <release-name>
  ```

- List all releases:
  ```bash
  helm list
  ```

- Show release status:
  ```bash
  helm status <release-name>
  ```

- Render chart templates locally:
  ```bash
  helm template <release-name> <chart>
  ```

- Lint a chart:
  ```bash
  helm lint <chart-path>
  ```

- Scaffold a new chart:
  ```bash
  helm create <chart-name>
  ```