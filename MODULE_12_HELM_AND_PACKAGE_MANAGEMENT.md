# Module 12: Helm and Package Management

## What is Helm?
Helm is the package manager for Kubernetes. Just as `apt` installs software packages on Ubuntu or `npm` installs JavaScript libraries, Helm installs and manages Kubernetes applications. A single Helm command can deploy a complete application stack — including Deployments, Services, ConfigMaps, Secrets, Ingresses, and more — all from a versioned, shareable package called a chart.

Without Helm, deploying a complex application to Kubernetes means maintaining a large collection of YAML files, manually tracking which versions are deployed in which environment, and carefully applying changes in the correct order. Helm handles all of this by treating a deployment as a named, versioned release that can be installed, upgraded, rolled back, and uninstalled as a unit.

Helm 2 required a server-side component called Tiller that ran inside the cluster and acted as an intermediary between the Helm client and the Kubernetes API. Tiller had significant security implications because it required broad cluster permissions. Helm 3, released in 2019, removed Tiller entirely. All operations now happen directly from the client using the user's own Kubernetes credentials and RBAC permissions.

## Helm Core Concepts

### Charts
A chart is a collection of files that describe a Kubernetes application. It contains YAML templates for every Kubernetes resource the application needs, a `values.yaml` file with default configuration, and a `Chart.yaml` file with metadata. Charts can be packaged into a `.tgz` archive and shared via a repository.

### Releases
A release is a named, running instance of a chart in a Kubernetes cluster. You can install the same chart multiple times in the same cluster under different release names — for example, installing a `postgresql` chart as both `db-staging` and `db-production` in different namespaces. Each release has its own history and can be upgraded or rolled back independently.

### Repositories
A chart repository is an HTTP server that hosts packaged chart archives and an index file. Helm clients pull charts from repositories. ArtifactHub is the central discovery platform for Helm charts from hundreds of publishers. Bitnami maintains one of the most popular community chart repositories, with production-grade charts for databases, message brokers, monitoring stacks, and more.

### Values
Values are user-supplied configuration that override the defaults in `values.yaml`. You can pass values on the command line with `--set key=value` or by providing a custom values file with `-f my-values.yaml`. Values flow into the chart templates through the `.Values` object.

## Installing and Using Helm

Add a repository and update the local cache:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Search for available charts:

```bash
helm search repo postgresql
```

Install a chart as a named release:

```bash
helm install my-postgres bitnami/postgresql --namespace databases --create-namespace
```

Upgrade a release to a new chart version or with changed values:

```bash
helm upgrade my-postgres bitnami/postgresql --set primary.persistence.size=20Gi
```

Roll back to a previous revision:

```bash
helm rollback my-postgres 1
```

Uninstall a release and clean up all its resources:

```bash
helm uninstall my-postgres --namespace databases
```

List all releases in the current namespace:

```bash
helm list
helm list --all-namespaces
```

Show the current status and last deployed time of a release:

```bash
helm status my-postgres
```

Override values at install or upgrade time using `--set` for individual values or `-f` for a complete values file:

```bash
helm install my-app ./my-chart --set replicaCount=3 --set image.tag=v2.1.0
helm install my-app ./my-chart -f production-values.yaml
```

## Chart Structure
A Helm chart is a directory with a specific layout:

```
my-chart/
├── Chart.yaml           # Chart metadata: name, version, appVersion, description
├── values.yaml          # Default configuration values
├── templates/           # Kubernetes YAML templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl     # Named templates and helper functions
│   └── NOTES.txt        # Post-install instructions printed to the user
└── charts/              # Dependency charts (subcharts)
```

Templates use the Go template language with Helm-specific functions. Values from `values.yaml` are referenced as `.Values.<key>`. Release metadata is accessible as `.Release.Name`, `.Release.Namespace`, and `.Release.IsInstall`. Chart metadata is available as `.Chart.Name` and `.Chart.Version`.

A typical Deployment template fragment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

Named templates are defined in `_helpers.tpl` using the `define` action and called with `include`. This keeps templates DRY by centralizing common label sets, name construction logic, and other reusable fragments.

## Creating Your Own Chart
Helm can scaffold a new chart directory with all the standard files:

```bash
helm create my-chart
```

This generates a sample chart with a Deployment, Service, Ingress, and ServiceAccount. Edit `values.yaml` to define your configuration parameters, then update the templates in `templates/` to use those values.

Render templates locally to inspect the output before deploying:

```bash
helm template my-release ./my-chart -f my-values.yaml
```

Validate the chart for common errors and YAML correctness:

```bash
helm lint ./my-chart
```

Package the chart into a `.tgz` archive for distribution:

```bash
helm package ./my-chart
```

You can publish the resulting archive to any HTTP server and point users to it as a Helm repository.

## Helm Hooks
Helm hooks are annotations that instruct Helm to run specific templates at particular points in the lifecycle of a release. Common hook annotations include:

- `helm.sh/hook: pre-install` — runs before any other resources are created during an install
- `helm.sh/hook: post-install` — runs after all resources are created
- `helm.sh/hook: pre-upgrade` — runs before upgrading resources
- `helm.sh/hook: post-upgrade` — runs after all upgraded resources are ready

A common pattern is to run a database migration Job as a `pre-upgrade` hook. This ensures the schema is updated before the new application version starts receiving traffic, preventing the new code from encountering an incompatible schema.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["./migrate.sh"]
```

## Kustomize as an Alternative
Kustomize is a configuration management tool built directly into kubectl (available since Kubernetes 1.14). Unlike Helm, Kustomize does not use a templating engine. Instead, it uses an overlay system where you start with a base set of YAML files and apply patches on top of them for each environment.

A Kustomize project has a `kustomization.yaml` file that references the base resources and any patches or overlays to apply:

```yaml
# overlays/production/kustomization.yaml
resources:
- ../../base
patchesStrategicMerge:
- replica-patch.yaml
images:
- name: my-app
  newTag: v2.1.0
```

Apply a Kustomize configuration directly with kubectl:

```bash
kubectl apply -k overlays/production/
```

Kustomize is a better fit than Helm when you prefer pure YAML without a templating language, when you want to layer environment-specific patches on top of base configurations, or when your application manifests are maintained by a team that prefers direct YAML over chart abstractions. Helm is generally more appropriate when you need to share reusable, configurable packages with external users or across many teams.

## Best Practices
Always pin chart versions when using third-party charts in production. Pulling `latest` or using floating version constraints means an unexpected chart update can break your deployment. Specify exact versions in your install commands or in your GitOps repository.

Store your custom `values.yaml` overrides in version control alongside your application code or in a dedicated configuration repository. This creates an auditable history of every configuration change and makes it easy to reproduce deployments in any environment.

Use the `helm-diff` plugin before running `helm upgrade` in production. The plugin shows exactly which Kubernetes resources will be created, modified, or deleted by the upgrade, catching unintended changes before they are applied.

Keep charts simple and composable. Avoid building monolithic charts that deploy an entire platform from a single `helm install` command. Smaller, focused charts are easier to test, upgrade, and reason about. Use chart dependencies (`charts/` directory) to compose them when needed.
