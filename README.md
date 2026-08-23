# dnscontrol-helm

Helm chart for [dnscontrol](https://github.com/SvarogLab/dnscontrol), a controller that converges
Google Cloud DNS to a directory of declarative YAML.

Published to GHCR as an OCI artifact. Note the path: `helm push` names the artifact after the chart,
so it lands one level below the repository, keeping it clear of the controller's container image at
`ghcr.io/svaroglab/dnscontrol`.

```bash
helm install dnscontrol oci://ghcr.io/svaroglab/dnscontrol-helm/dnscontrol --version 0.1.0 \
  --set config.existingConfigMap=dns-zones \
  --set gcp.project=my-project
```

## What the chart does and does not do

It deploys the controller: a Deployment, a ServiceAccount, and a mount of a ConfigMap **you**
provide. It never creates that ConfigMap and never creates a Secret — your zone data and your
credentials are not the chart's to own, and keeping them out is what lets this repository stay
public and free of real data.

> The controller is authoritative over the whole GCP project: any managed zone not declared in the
> config is deleted. Point it at a project you own exclusively, and keep `--check` in `args` until
> you have watched a real diff go by.

For ArgoCD's ApplicationSet:

```yaml
# .appset.yaml
chart: dnscontrol
chartRepoURL: ghcr.io/svaroglab/dnscontrol-helm
targetRevision: "0.1.0"
releaseName: dnscontrol
namespace: dns
```

## Values

| Key | Default | Notes |
| --- | --- | --- |
| `image.repository` | `ghcr.io/svaroglab/dnscontrol` | |
| `image.tag` | `.Chart.AppVersion` | An immutable `vX.Y.Z`; `latest` is rejected at render time |
| `imagePullSecrets` | `[]` | Required if the GHCR package is private — see below |
| `config.existingConfigMap` | `dns-zones` | Required; the chart fails to render without it |
| `config.mountPath` | `/etc/dnscontrol` | |
| `gcp.project` | `""` | Empty means the key, then the metadata server |
| `gcp.existingSecret` | `""` | Empty means Workload Identity — nothing is mounted |
| `gcp.secretKey` | `key.json` | |
| `args` | `["--watch", "--diff"]` | Add `--check` to make every run read-only |
| `logLevel` | `info` | |
| `serviceAccount.annotations` | `{}` | `iam.gke.io/gcp-service-account` for Workload Identity |

### Pulling the image

A GHCR package is **private on first push, even from a public repository** — GHCR does not inherit
repository visibility. Either make the package public once, under
`https://github.com/orgs/<org>/packages/container/package/dnscontrol` → Package settings → Change
visibility, or leave it private and name a pull secret:

```yaml
imagePullSecrets:
  - name: ghcr-credentials
```

## Fixed by the template, not configurable

- **`replicas: 1`, `strategy: Recreate`.** Two replicas would race on the same Cloud DNS changes,
  and the loser's deletions would no longer match what is stored, failing its whole atomic change.
  There is no leader election by design.
- **The ConfigMap is mounted as a volume, never with `subPath`.** A `subPath` mount does not receive
  ConfigMap updates at all, which would leave the controller's watch permanently blind.
- **`terminationGracePeriodSeconds: 60`**, comfortably above the controller's own 45-second ceiling
  on waiting for a Cloud DNS change, so `SIGTERM` never lands mid-change.
- **`runAsNonRoot`, `readOnlyRootFilesystem`, all capabilities dropped.** The image is `FROM
  scratch` and writes nothing.

## Development

```bash
helm lint .
helm lint . -f .ci/example-values.yaml
helm template ci . && helm template ci . -f .ci/example-values.yaml
```

Everything committed here uses reserved example names only.
