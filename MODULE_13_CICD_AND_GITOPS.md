# Module 13: CI/CD and GitOps with Kubernetes

## CI/CD for Kubernetes Overview
Continuous Integration and Continuous Delivery (CI/CD) is the practice of automating the build, test, and deployment pipeline so that code changes reach production quickly and reliably. In a traditional server-based world, deployment often meant copying files to a server or running an Ansible playbook. With Kubernetes, deployment means publishing a new container image and updating a Kubernetes manifest to reference it.

The typical pipeline for a Kubernetes workload looks like this:

1. A developer pushes code to a Git repository.
2. The CI system builds the application and runs automated tests.
3. On success, the CI system builds a Docker image and pushes it to a container registry.
4. The pipeline updates the Kubernetes manifests (or Helm values) with the new image tag.
5. The manifests are applied to the Kubernetes cluster, triggering a rolling update.

The tools most commonly used at each step include GitHub Actions, GitLab CI, Jenkins, and Tekton for the CI pipeline, and ArgoCD or Flux for the delivery and synchronization step.

## GitHub Actions for Kubernetes
GitHub Actions is a popular CI/CD platform that runs workflows defined as YAML files in the `.github/workflows/` directory of a repository. For Kubernetes deployments, a typical workflow builds a Docker image, pushes it to a registry, and then either directly applies manifests to the cluster or updates a manifests repository that a GitOps tool like ArgoCD is watching.

To interact with a Kubernetes cluster from a GitHub Actions workflow, you store the cluster's `KUBECONFIG` contents as a GitHub repository secret. The workflow then writes this secret to a file and sets `KUBECONFIG` to point to it before running `kubectl` or `helm` commands.

Example GitHub Actions workflow:

```yaml
name: Deploy to Kubernetes

on:
  push:
    branches:
    - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Check out code
      uses: actions/checkout@v4

    - name: Log in to container registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        push: true
        tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

    - name: Set up kubectl
      uses: azure/setup-kubectl@v3

    - name: Deploy to cluster
      env:
        KUBECONFIG_DATA: ${{ secrets.KUBECONFIG }}
      run: |
        echo "$KUBECONFIG_DATA" | base64 -d > /tmp/kubeconfig
        export KUBECONFIG=/tmp/kubeconfig
        kubectl set image deployment/api-server \
          api-server=ghcr.io/${{ github.repository }}:${{ github.sha }}
        kubectl rollout status deployment/api-server
```

This workflow triggers on every push to the `main` branch. It builds and pushes the Docker image tagged with the Git commit SHA, then updates the Deployment image and waits for the rollout to complete. Using the commit SHA as the image tag ensures every image is uniquely and traceably identified.

## GitOps Principles
GitOps is an operational model that uses Git as the single source of truth for both application code and infrastructure configuration. The core principles are:

**Declarative infrastructure**: Everything that describes the desired state of the system — Kubernetes manifests, Helm values, Kustomize overlays — is expressed declaratively and stored in Git.

**Version-controlled and auditable**: Every change to the system passes through a Git commit. This creates a complete, immutable audit trail. You can answer the question "what changed, when, and who approved it?" by reading the Git history.

**Automated reconciliation**: A GitOps agent running inside the cluster continuously compares the desired state in Git against the actual state in the cluster. When a drift is detected — because someone applied a manifest manually, or a node failed and a Pod was rescheduled with different settings — the agent automatically reconciles the cluster back to the state defined in Git.

**Push-based vs Pull-based deployment**: Traditional CI/CD pipelines push changes to the cluster by running `kubectl apply` from a pipeline runner outside the cluster. GitOps uses a pull-based model where an agent inside the cluster pulls changes from Git and applies them. The pull-based model is more secure because the cluster credentials never need to leave the cluster.

## ArgoCD
ArgoCD is the most widely adopted GitOps continuous delivery tool for Kubernetes. It runs inside the cluster as a set of Deployments and provides both a web UI and a CLI for managing application delivery.

### How ArgoCD Works
ArgoCD is configured with an `Application` custom resource that points to a Git repository path and a target cluster and namespace. ArgoCD continuously polls the Git repository (or receives webhook notifications) and compares the manifests in Git against the live state of the cluster. When they diverge, ArgoCD can either notify you (manual sync mode) or automatically reconcile (auto-sync mode).

### ArgoCD Application Manifest
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-server
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests
    targetRevision: main
    path: apps/api-server/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

The `automated` sync policy enables auto-sync. The `prune: true` flag means ArgoCD will delete resources that exist in the cluster but are no longer in Git. The `selfHeal: true` flag means ArgoCD will immediately re-apply the Git state if someone manually modifies a resource in the cluster.

### App of Apps Pattern
For managing many applications, ArgoCD supports the App of Apps pattern. A single root `Application` points to a directory containing other ArgoCD `Application` manifests. ArgoCD deploys the root App, which in turn causes it to discover and deploy all child Apps. This allows you to bootstrap an entire cluster's worth of applications from a single `kubectl apply`.

### ArgoCD CLI Commands
```bash
argocd app list
argocd app sync api-server
argocd app status api-server
argocd app history api-server
argocd app rollback api-server 3
```

### Health Checks and Sync Status
ArgoCD reports two independent statuses for every application: sync status (whether the cluster matches Git) and health status (whether the Kubernetes resources are in a healthy state). It understands the health semantics of built-in Kubernetes resources — a Deployment is healthy when all its replicas are ready, a Job is healthy when it has completed. Custom health checks can be written in Lua for custom resources.

## Flux CD
Flux is an alternative GitOps tool that takes a more modular, toolkit-based approach. Rather than a single monolithic application, Flux is composed of independent controllers — the Source Controller, Kustomize Controller, Helm Controller, and Image Automation Controller — each of which can be used independently.

Flux uses its own CRDs to represent Git sources, Kustomize configurations, and Helm releases:

- `GitRepository`: Defines a Git repo and branch to sync from.
- `Kustomization`: Defines a path within a GitRepository to apply with Kustomize.
- `HelmRelease`: Defines a Helm chart release that Flux manages and keeps in sync.

Flux is often preferred by teams that want fine-grained control over the individual components of their GitOps stack, or who already use Kustomize and want native integration. ArgoCD is often preferred by teams that want a single tool with a strong UI and a broad ecosystem of integrations.

## Image Update Automation
A common challenge in GitOps is keeping image tags in Git up to date when new images are published. If your CD pipeline updates the Deployment directly via `kubectl set image`, the Git repository falls out of sync with the cluster.

Both ArgoCD and Flux provide image update automation that solves this problem:

- **Flux Image Automation**: The Flux Image Automation Controller watches a container registry for new image tags matching a policy (e.g., semver `~1.2`), writes the new tag back to the Git repository as a commit, and then the Kustomize Controller picks up the change and applies it to the cluster.
- **ArgoCD Image Updater**: Works similarly, scanning registries and writing updated image tags back to the GitOps repository.

This closes the loop: the Git repository always reflects what is actually running, and all changes go through Git.

## Pipeline Security
Modern Kubernetes CI/CD pipelines operate at the intersection of application code, container images, and infrastructure configuration, making them a high-value target for supply chain attacks. Several practices are essential:

**Image signing with Cosign**: Cosign (from the Sigstore project) allows you to cryptographically sign container images and verify those signatures in your deployment pipeline or with a policy engine. This ensures that only images built by your trusted CI system are deployed to production.

**Vulnerability scanning with Trivy**: Trivy is an open-source vulnerability scanner that can be integrated into your CI pipeline to scan container images for known CVEs. A failed scan can block promotion of an insecure image to production.

**Policy enforcement with OPA Gatekeeper or Kyverno**: These tools enforce admission policies in the cluster. For example, you can enforce that all images must come from your trusted registry, that all containers must set resource limits, or that no container may run as root. Gatekeeper uses Rego policies; Kyverno uses a YAML-native policy language. Both can also be run in CI pipelines to validate manifests before they are ever applied to the cluster.

## Best Practices
Keep application source code and application configuration (Kubernetes manifests, Helm values) in separate Git repositories. The source repo triggers builds and image pushes; the config repo is the source of truth for what is deployed. This separation prevents a code change from bypassing the delivery review process.

Use environment branches or directories within your config repository to represent dev, staging, and production environments. A common pattern is a directory structure such as `apps/<service>/overlays/dev`, `apps/<service>/overlays/staging`, and `apps/<service>/overlays/production`, where each overlay layer adds environment-specific configuration on top of a shared base.

Never apply Kubernetes manifests manually in production. Every change should flow through Git, be reviewed via a pull request, and be applied by the GitOps agent. Manually applied changes are immediately overwritten by the reconciliation loop and leave no audit trail.

Use a secret management solution that integrates with GitOps rather than storing raw secrets in Git. Options include the External Secrets Operator (which reads secrets from AWS Secrets Manager, HashiCorp Vault, or GCP Secret Manager and creates Kubernetes Secrets automatically), Sealed Secrets (which encrypts secrets using a cluster-held key so the encrypted form is safe to commit), and SOPS (Mozilla Secrets OPerationS, which encrypts files using age or PGP keys).

Enforce image digest pinning in production. Image tags are mutable — the same tag can point to a different image after a new push. Pinning to a digest (e.g., `my-app@sha256:abc123...`) guarantees that exactly the image you tested is what gets deployed, and no one can silently replace it.
