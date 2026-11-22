# KubeWatcher charts

The Helm chart of the [KubeWatcher](https://kubewatcher.com) agent, and nothing else. The agent runs in your cluster, watches it read-only and streams what it sees to KubeWatcher over an outbound mTLS connection. Its source code is not in this repository: the agent is a closed-source image (`docker.io/aaraz/kubewatcher-cluster-agent`) and this repository holds only the chart that installs it.

The chart is published as an OCI chart at `oci://registry-1.docker.io/aaraz/cluster-agent`. `charts/cluster-agent/` here is the same chart, so you can read exactly what it installs (its Deployment, its Roles and its network policy) before you run it.

## Install

Add a cluster in the KubeWatcher dashboard. It shows the install command with your single-use enrollment token already filled in. The shape of it is:

```bash
helm install kubewatcher-agent oci://registry-1.docker.io/aaraz/cluster-agent --version <version> \
  --namespace kubewatcher-system --create-namespace \
  -f https://<your KubeWatcher address>/v1/enroll/<token>/values.yaml
```

The values file served by that link carries the cluster name, the gateway address, the enrollment token and the gateway's CA certificate. Treat the link as a secret: it works once. The chart's own README (`charts/cluster-agent/README.md`) lists every value.

## Upgrade

```bash
helm upgrade kubewatcher-agent oci://registry-1.docker.io/aaraz/cluster-agent --version <version> \
  --namespace kubewatcher-system --reset-then-reuse-values
```

`--reset-then-reuse-values` applies the new chart's defaults and keeps the values you set yourself, where `--reuse-values` would keep the old chart's defaults and miss the new ones. It needs Helm 3.14 or newer.

## What changed

- Every release has a page under [Releases](https://github.com/kubewatcher/charts/releases) with its notes, and a compare link between it and the release before it.
- `CHANGELOG.md` lists the same notes in one file.
- `releases.json` is the machine-readable list (version, date, severity, whether the Kubernetes permissions changed, a summary and the links). KubeWatcher reads it to tell you when a new version is out. Its format is `releases.schema.json`.

## Updates by email

On GitHub, choose Watch, then Custom, then Releases. Or follow the Atom feed at `https://github.com/kubewatcher/charts/releases.atom` in a feed reader.

## Support

The latest two minor versions receive fixes. Older versions keep working but are not updated.

## Security

See [SECURITY.md](SECURITY.md).
