# Module 7: Security

## Why Security Matters in Kubernetes
Kubernetes clusters run workloads that often handle sensitive data, and a misconfigured cluster can expose applications to serious risks. Security in Kubernetes is a multi-layered concern spanning authentication, authorization, network isolation, and secrets management. Understanding these layers is essential for anyone operating Kubernetes in production.

## Authentication
Authentication is the process of verifying who is making a request to the Kubernetes API Server. Kubernetes supports several authentication strategies, and multiple can be active at once. Every request to the API Server must be authenticated before it is authorized.

Authentication methods:
- **X.509 Client Certificates**: The most common method for cluster components and admin users. A certificate signed by the cluster CA proves identity.
- **Bearer Tokens**: Used by ServiceAccounts and external identity providers. Tokens are passed in the Authorization header.
- **OIDC (OpenID Connect)**: Integrates Kubernetes with external identity providers like Google, Azure AD, or Okta for user authentication.
- **ServiceAccount Tokens**: Automatically mounted into Pods, allowing workloads to authenticate to the API Server.

## Authorization with RBAC
Role-Based Access Control (RBAC) is the standard authorization mechanism in Kubernetes. It controls what authenticated users and ServiceAccounts are allowed to do. RBAC uses four key objects to define and assign permissions.

RBAC objects:
- **Role**: Defines a set of permissions within a specific namespace (e.g., get/list/watch Pods)
- **ClusterRole**: Like a Role but applies cluster-wide, or can be used to grant access to non-namespaced resources
- **RoleBinding**: Binds a Role to a user, group, or ServiceAccount within a namespace
- **ClusterRoleBinding**: Binds a ClusterRole to a user, group, or ServiceAccount cluster-wide

Key RBAC commands:
- kubectl get roles — list roles in the current namespace
- kubectl get clusterroles — list cluster-wide roles
- kubectl get rolebindings — list role bindings in the current namespace
- kubectl auth can-i get pods --as=user — check if a user has permission to perform an action
- kubectl auth can-i --list — list all actions the current user can perform

## ServiceAccounts
A ServiceAccount provides an identity for processes running inside a Pod. Every Pod is automatically assigned the default ServiceAccount of its namespace unless specified otherwise. ServiceAccounts are used to grant Pods the minimum necessary permissions to interact with the Kubernetes API.

ServiceAccount best practices:
- Create dedicated ServiceAccounts for each application rather than using the default
- Use RBAC RoleBindings to grant only the permissions each ServiceAccount needs
- Disable auto-mounting of ServiceAccount tokens when not needed (automountServiceAccountToken: false)
- Rotate ServiceAccount tokens regularly in sensitive environments

## Secrets Management
Secrets store sensitive information such as passwords, API keys, and TLS certificates. By default, Secrets are base64-encoded (not encrypted) in etcd. For production security, enable encryption at rest and consider using an external secrets manager.

Best practices for Secrets:
- Enable etcd encryption at rest using EncryptionConfiguration
- Use tools like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault for external secret management
- Use the External Secrets Operator to sync external secrets into Kubernetes Secrets
- Avoid storing Secrets in source control or Docker images
- Mount Secrets as volumes rather than environment variables where possible (env vars can be logged)

## Pod Security
Pod security controls restrict what a container can do at the OS level. Kubernetes previously used PodSecurityPolicies (deprecated in 1.21, removed in 1.25) and now uses Pod Security Admission (PSA) with built-in security standards.

Pod Security Standards:
- **Privileged**: No restrictions. Only for trusted system workloads.
- **Baseline**: Minimally restrictive. Prevents known privilege escalations.
- **Restricted**: Heavily restricted. Follows current pod hardening best practices.

Security context settings for Pods and containers:
- runAsNonRoot: true — prevents containers from running as root
- readOnlyRootFilesystem: true — makes the container filesystem read-only
- allowPrivilegeEscalation: false — prevents a process from gaining more privileges than its parent
- capabilities — drop all Linux capabilities and add only what is needed

## Network Security with Network Policies
By default, all Pods in a cluster can communicate freely. Network Policies allow you to define rules that restrict Pod-to-Pod and Pod-to-external communication. They are enforced by the CNI plugin and are namespace-scoped.

Key network security patterns:
- Default-deny all ingress and egress traffic in a namespace, then whitelist only what is needed
- Isolate namespaces from each other using namespaceSelector rules
- Restrict database Pods to only accept connections from application Pods
- Use egress rules to prevent Pods from making unexpected outbound connections

## Image Security
Container images are a common attack vector. Using untrusted or unpatched images can introduce vulnerabilities into your cluster. Implement image security controls to reduce this risk.

Image security practices:
- Use minimal base images (e.g., distroless, Alpine) to reduce attack surface
- Scan images for vulnerabilities using tools like Trivy, Snyk, or Anchore
- Use image pull policies and private registries to control which images can run
- Sign images with Cosign (part of the Sigstore project) and enforce signatures with admission controllers
- Never run containers as root; use a non-root USER in your Dockerfile

## Admission Controllers
Admission controllers are plugins that intercept API Server requests after authentication and authorization, but before the object is persisted. They can validate or mutate incoming requests, enforcing security policies cluster-wide.

Common admission controllers:
- **NamespaceLifecycle**: Prevents creation of resources in terminating namespaces
- **LimitRanger**: Enforces resource limits on Pods
- **PodSecurity**: Enforces Pod Security Standards
- **MutatingAdmissionWebhook / ValidatingAdmissionWebhook**: Custom webhooks for policy enforcement (used by tools like OPA Gatekeeper and Kyverno)

## Why Security Cannot Be an Afterthought
Security in Kubernetes must be designed in from the start, not added later. A single misconfigured RBAC binding, an unpatched image, or a missing Network Policy can expose your entire cluster. Following the principle of least privilege, keeping components updated, and regularly auditing your cluster's security posture are the foundations of a secure Kubernetes environment.