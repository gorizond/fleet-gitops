# fleet-gitops

This repository is a Rancher Fleet “bootstrap” GitOps repo. It contains a Helm chart that provisions Fleet `GitRepo` (and optional `HelmOp`) resources, which Fleet then uses to deploy other GitOps repositories (and Helm workloads) across your clusters.

## How it works

1. You register this repo in Fleet (typically as a `GitRepo` in the `fleet-local` namespace on the management cluster).
2. Fleet detects the Helm chart (`Chart.yaml`) and renders it.
3. The chart creates downstream `fleet.cattle.io` custom resources (`GitRepo`/`HelmOp`) from `values.yaml`.
4. Fleet reconciles those resources and deploys the referenced repos/charts to clusters selected via labels/selectors.

## Configure downstream repos

Edit `values.yaml`:

- `gitrepos`: list of Fleet `GitRepo` resources to create.
- `helmrepos`: list of Fleet `HelmOp` resources to create (optional).

### `gitrepos[]` fields

- `name` (required): resource name and default cluster-label key.
- `namespace` (optional, default `fleet-default`): use `fleet-local` for local-only repos.
- `description`, `labels` (optional): metadata helpers.
- Repository URL parts (optional):
  - `repo_domain` (default `https://github.com`)
  - `repo_path` (default `gorizond`)
  - `repo` (default `<name>`)
- Other optional Fleet fields: `targetNamespace`, `clientSecretName`, `imageScanInterval`, `imageScanCommit`, `ociRegistrySecret`, `keepResources`.
- Targeting helpers (optional):
  - `clusterGroups`: list of Fleet cluster groups
  - `clusterSelector`: label map for an additional selector target
  - `targets`: list of additional label keys (each expects `<key>=enabled`)

### `helmrepos[]` fields

- `name` (required): resource name and default cluster-label key.
- `namespace` (optional, default `fleet-default`): use `fleet-local` for local-only HelmOps.
- `description`, `labels` (optional): metadata helpers.
- `helm` (required): YAML object rendered into `HelmOp.spec.helm` (for example: `repo`, `chart`, `releaseName`, `version`, `values`).
- `dependsOn` (optional): YAML list rendered into `HelmOp.spec.dependsOn`.
- `targetNamespace` (optional): rendered into `HelmOp.spec.namespace`.
- Targeting helpers (optional): `clusterGroups`, `clusterSelector`, `targets` (same behavior as `gitrepos[]`).

### Cluster targeting (labels)

For every downstream entry that is **not** in `fleet-local`, the chart always adds a target selector matching clusters labelled:

- `<name>=enabled`

Example: to enable a downstream repo named `fleet-cert-manager` on a cluster, add the label `fleet-cert-manager=enabled` to that cluster in Rancher.

## Bootstrap / install

Register this repository in Fleet (Rancher UI) or apply a `GitRepo` manifest like:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: fleet-gitops
  namespace: fleet-local
spec:
  repo: https://github.com/<org-or-user>/fleet-gitops
  branch: main
  paths:
    - .
```

After it reconciles, Fleet will create the downstream `GitRepo`/`HelmOp` resources defined in `values.yaml`.

## Render locally (optional)

```sh
helm template fleet-gitops . -f values.yaml
```

## Notes

- Downstream `GitRepo.spec.branch` is currently set to `main` in `templates/git-repos.yaml`.
