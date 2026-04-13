# Introduction to Kubernetes

## What is Kubernetes?
Kubernetes is an open-source container orchestration platform designed for automating the deployment, scaling, and management of containerized applications. It allows developers to manage microservices in a predictable and efficient manner.

## Key Characteristics
1. **Self-healing**: Automatically replaces and reschedules containers from failed nodes.
2. **Horizontal scaling**: Scales applications up and down easily and quickly.
3. **Service discovery and load balancing**: Automatically exposes containers using DNS names or using their own IP addresses.
4. **Automated rollouts and rollbacks**: Gradually roll out changes to applications while monitoring application health.
5. **Secret and configuration management**: Helps manage sensitive information and configuration separate from application code.

## Kubernetes Architecture
### Master/Control Plane Components
- **API Server**: Serves the Kubernetes API using JSON over HTTP, acts as the frontend for the control plane.
- **Scheduler**: Assigns work to individual nodes by prioritizing them based on resource availability.
- **Controller Manager**: Regulates the state of the cluster by ensuring that the desired state matches the actual state.
- **etcd**: A key-value store used for storing all cluster data, serving as a reliable way to manage the cluster state.

### Worker Node Components
- **Kubelet**: An agent that runs on each worker node, communicating with the control plane and ensuring that containers are running in pods.
- **Kube-proxy**: Manages network communication, allowing pods to communicate with each other and with the outside world.
- **Container Runtime**: Software responsible for running containers (e.g., Docker, containerd).

## Kubernetes Objects
Kubernetes objects are persistent entities in the cluster that represent the state of your application. Some important objects are:
- **Pod**: The smallest deployable unit, which can contain one or more containers.
- **Service**: An abstraction that defines a logical set of pods and a policy to access them.
- **Deployment**: Provides declarative updates to pods, allowing for scaling or rolling updates.

## Installation Methods
1. **Minikube**: A tool that makes it easy to run Kubernetes locally.
2. **Kubeadm**: A toolkit that helps you bootstrap a best-practices Kubernetes cluster.
3. **Managed Kubernetes Services**: Cloud providers like AWS (EKS), Azure (AKS), and Google Cloud (GKE) provide managed Kubernetes services.

## Why Kubernetes?
- It simplifies the development and management of containerized applications.
- It enhances scalability and efficiency of app deployment.
- It provides a unified approach to managing microservices in dynamic environments.