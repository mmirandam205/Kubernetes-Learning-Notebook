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