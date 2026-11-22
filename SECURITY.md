# Security

## Reporting a vulnerability

Please do not open a public issue for a vulnerability. Use this repository's private vulnerability reporting: open the Security tab, choose Report a vulnerability, and include what you found, how to reproduce it and which chart version you used. Only the maintainers see the report. We acknowledge a report and keep you told of the fix.

## In scope

- The chart in `charts/cluster-agent/`: the roles and bindings it creates, the Pod it runs, its network policy, its Secrets and the way it handles your enrollment token.
- The agent image that the chart installs (`docker.io/kubewatcher/cluster-agent`) and the connection it makes to KubeWatcher.
- The release files in this repository (`releases.json`).

## Read-only by default

The agent is read-only unless you change that. With the default `mode: read-only` the chart installs no write permissions and the agent refuses every change. The agent mode and its permissions are set only in Helm values by whoever installs the chart; KubeWatcher cannot change them from the backend. Remediation mode is a separate opt-in that adds its own role.

Releases that fix a vulnerability are marked `security` in `releases.json`, and releases that change the Kubernetes permissions the chart asks for are marked `rbac_changed`.
