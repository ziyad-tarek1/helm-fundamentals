
# Helm Study Notes

## Helm2 vs Helm3

### Tiller in Helm2
- **Tiller Component**: 
  - Helm2 relied on a server-side component called **Tiller** deployed in the cluster.
  - Tiller was responsible for managing releases (installing/upgrading/deleting charts).
  - **Security/RBAC Issues**:
    - Tiller required broad permissions in the cluster, often needing `cluster-admin` rights.
    - This led to security concerns, as Tiller's permissions were not scoped to specific namespaces or roles.
    - RBAC (Role-Based Access Control) configurations were complex and error-prone.
- **CRDs (Custom Resource Definitions)**:
  - Helm2 did not natively support CRDs as first-class citizens. CRD management required manual steps or hooks.

### Helm3: Removing Tiller
- **Tiller Removal**:
  - Helm3 eliminates Tiller, moving all logic to the client-side (`helm` CLI).
  - **Benefits**:
    - Simplified security model: Permissions align with the user's kubeconfig credentials.
    - No more RBAC challenges tied to Tiller.
- **CRD Handling**:
  - Helm3 treats CRDs as standard Kubernetes resources, allowing them to be managed directly in charts (with safeguards to prevent accidental deletion).

---

## 3-Way Strategic Merge Patch
- **What is it?**
  - Helm3 uses a **3-way merge strategy** during upgrades:
    1. **Last Applied Configuration** (the original state of the resource).
    2. **Current Live State** (the live state in the cluster).
    3. **New Changes** (the desired state from the updated chart).
  - This resolves conflicts more effectively than Helm2’s 2-way merge.
- **Why it matters**:
  - Reduces unexpected changes or data loss during upgrades.
  - Aligns with Kubernetes' native `kubectl apply` behavior for strategic merging.

---

## Helm Metadata Storage
- **Release Metadata**:
  - Helm stores metadata (e.g., release history, chart versions, values) **as Kubernetes Secrets** in the cluster.
  - **Location**: Secrets are created in the same namespace as the release.
- **Structure**:
  - Secrets are labeled with:
    - `owner: helm`
    - `status: deployed` (or `superseded`/`failed`).
  - Example Secret name: `sh.helm.release.v1.<release-name>.v1`.
- **Benefits**:
  - **Accessibility**: Secrets are accessible to all team members with namespace access.
  - **Security**: Secrets are encrypted at rest (if the cluster is configured for encryption).
  - **Auditability**: Full release history is stored and recoverable.
- **Helm2 vs Helm3**:
  - Helm2 stored release metadata in **ConfigMaps** (less secure).
  - Helm3’s use of Secrets aligns with better security practices.
```

### Example: Viewing Helm Secrets
```bash
# List all Helm secrets in a namespace
kubectl get secrets -l owner=helm -n <namespace>
```

---

# Helm Repositories, Charts, Releases & Common Commands

---

## Helm Repositories
### What is a Helm Repository?
- A **Helm repository** is a collection of Helm charts stored remotely (e.g., HTTP server, OCI registry) or locally.
- **Default Repositories**:
  - **Artifact Hub** (`https://artifacthub.io`): The primary public hub for Helm charts (replaced Helm Hub).
  - Others: Organizations often host private repositories (e.g., ChartMuseum, GitHub Pages, S3 buckets).

### Repository Storage Locations
- **Local Configuration**: Helm repositories are stored in your local Helm configuration (at `~/.config/helm/repositories.yaml`).
- **Example Repositories**:
  ```bash
  # Add the Bitnami repository
  helm repo add bitnami https://charts.bitnami.com/bitnami
  # List configured repositories
  helm repo list
  # Update local repo metadata
  helm repo update
  ```

---

## Charts vs Releases
### Charts
- **Definition**: A Helm chart is a packaged application containing Kubernetes manifests, dependencies, and metadata.
- **Structure**: A folder with files like:
  ```
  mychart/
  ├── Chart.yaml
  ├── values.yaml
  ├── templates/
  └── charts/ (dependencies)
  ```
- **Example**: `bitnami/wordpress` is a chart for deploying WordPress.

### Releases
- **Definition**: A **release** is a deployed instance of a chart with specific configurations.
- **Example**: Installing `bitnami/wordpress` with the release name `my-site` creates a unique release in the cluster.

---

![alt text](image.png)

## Most Used Helm Commands

### Installation
```bash
# Install a chart from a repo
helm install my-release bitnami/nginx

# Install from a local chart directory
helm install my-release ./mychart

# Install with a values file
helm install my-release bitnami/wordpress -f custom-values.yaml

# Install with inline overrides
helm install my-release bitnami/wordpress --set service.type=NodePort
```

### Upgrading Releases
```bash
# Upgrade a release with new values
helm upgrade my-release bitnami/wordpress -f updated-values.yaml

# Override specific values during upgrade
helm upgrade my-release bitnami/wordpress --set replicaCount=3

# Rollback to a previous version
helm rollback my-release 2

```

### Managing Repositories
```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# List repositories
helm repo list

# Update repository metadata
helm repo update

# Search for charts in Artifact Hub
helm search hub wordpress

# Search in your configured repositories
helm search repo nginx
```

### Managing Releases
```bash
# List deployed releases
helm list

# Uninstall a release
helm uninstall my-release

# View release status
helm status my-release

# View release history
helm history my-release
```

---

## Example Workflow
1. Add a repository:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   ```
2. Install WordPress:
   ```bash
   helm install my-site bitnami/wordpress \
     --set service.type=LoadBalancer \
     --set mariadb.auth.rootPassword=secret
   ```
3. Upgrade with a values file:
   ```bash
   helm upgrade my-site bitnami/wordpress -f production-values.yaml
   ```
4. Clean up:
   ```bash
   helm uninstall my-site
   ```

---

To download the default `values.yaml` file of a specific Helm chart, use the `helm show values` command. Here's how:

---

### Command
```bash
helm show values [REPO_NAME]/[CHART_NAME] > values.yaml
```

### Example
```bash
# Download the default values for the `bitnami/wordpress` chart
helm show values bitnami/wordpress > wordpress-values.yaml
```

---

### Notes
1. **Prerequisite**: Ensure the repository is added and updated first:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   ```
2. **For Local Charts**: If you have the chart locally (e.g., `./mychart`), use:
   ```bash
   helm show values ./mychart > values.yaml
   ```
3. **For Charts in OCI Registries** (Helm 3.8+):
   ```bash
   helm show values oci://registry-1.docker.io/bitnamicharts/wordpress > values.yaml
   ```

---

### Alternative: Download the Entire Chart
If you want the entire chart (including `values.yaml`):
```bash
# Pull the chart as a .tgz file
helm pull bitnami/wordpress --untar
```
This creates a `wordpress` directory with the chart's files, including `values.yaml`.

---

