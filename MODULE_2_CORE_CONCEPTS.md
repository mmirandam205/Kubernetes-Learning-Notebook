# Core Concepts in Kubernetes

## Pods
A Pod is the smallest deployable unit in Kubernetes, which can contain one or more containers. Pods share the same network namespace, and they can communicate with each other through localhost.

## Deployments
A Deployment provides declarative updates to Pods and ReplicaSets. It allows you to define the desired state of your application, and Kubernetes will manage the deployment to achieve that state.

## Services
A Service in Kubernetes is an abstraction that defines a logical set of Pods and a policy by which to access them. It provides stable network identities and load balancing for the Pods.

## Labels
Labels are key-value pairs attached to objects such as Pods, Services, and Deployments that can be used to organize and select subsets of objects.

## Namespaces
Namespaces are a way to divide cluster resources between multiple users or teams. They enable the creation of multiple environments within the same cluster, like development, testing, and production.

## ConfigMaps
ConfigMaps are used to store configuration data in key-value pairs, which can be consumed by Pods as environment variables or as configuration files in a volume.

## Secrets
Secrets are a special type of ConfigMap intended to hold sensitive data such as passwords, OAuth tokens, and SSH keys. Secrets are base64 encoded and can be mounted into Pods as volumes or exposed as environment variables.

## ReplicaSets
A ReplicaSet ensures that a specified number of Pod replicas are running at any given time. It is commonly used as a part of a Deployment.

## StatefulSets
StatefulSets are used to manage the deployment and scaling of a set of Pods, and they offer guarantees about the ordering and uniqueness of these Pods, making them ideal for stateful applications.

## DaemonSets
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As Nodes are added to the cluster, Pods are added to them automatically, ensuring consistent behavior across the cluster.