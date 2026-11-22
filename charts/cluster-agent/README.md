# cluster-agent chart

Installs the KubeWatcher agent: it watches the cluster **read-only**, streams signals to your KubeWatcher backend over an
outbound mTLS gRPC connection, and runs signed, typed tasks.

```bash
helm install kubewatcher-agent oci://registry-1.docker.io/kubewatcher/cluster-agent-chart --version 0.2.1 \
  -n kubewatcher-system --create-namespace \
  -f https://kubewatcher.com/v1/enroll/<single-use token from the dashboard>/values.yaml
```

The dashboard prints this command with the link filled in. The link serves `clusterName`, `backendAddr`, `enrollmentToken` and `backendCA` (the CA that signed the gateway's certificate) as a values file, so you can also write them yourself with `--set` and `--set-literal`. The image is `docker.io/kubewatcher/cluster-agent` at the chart's `appVersion` (override with `image.*`; private registries use `imagePullSecrets`). The enrollment token is single use. After the agent enrolls, its identity lives in the `kubewatcher-agent-identity` Secret in
its own namespace and the token is no longer needed (you can remove it from your values).

## Permissions

| Role | When | What |
| --- | --- | --- |
| `kubewatcher-agent-read` (ClusterRole) | always | `get/list/watch` on pods, events, nodes, namespaces, persistentvolumeclaims, services, deployments, replicasets, statefulsets, daemonsets, jobs, horizontalpodautoscalers, endpointslices, ingresses; `get` on `pods/log`. **No secrets, no write verbs.** |
| `kubewatcher-agent-identity` (Role) | always | create, and get/update on the one Secret `kubewatcher-agent-identity`, in the agent's own namespace |
| `kubewatcher-agent-remediation` | `mode=remediate` | generated from the action handlers, for the actions switched on: `patch` on `deployments`, `get/patch/update` on `deployments/scale`. A Role per namespace if `remediation.namespaces` is set |

`mode` is `read-only` by default (`read-only` or `remediate`; anything else fails the install). In read-only mode the agent
registers no mutating handler and rejects any mutating task with `REJECTED_READ_ONLY_MODE`. Every mutating task additionally needs a signed grant and a human approval in the dashboard.

In `remediate` mode the actions switched on in `remediation.actions` (all four by default) get their permissions. `remediation.namespaces` is an optional hard limit that gives a Role per namespace instead of a cluster-wide role. Normally leave it empty: which namespaces the agent watches and may change is the watch scope, set in the dashboard. `remediation.maxConcurrentTasks` (2), `cooldown` (5m) and `bounds.*` are enforced by the agent whatever the backend sends.

## Values worth knowing

| Value | Default | What it does |
| --- | --- | --- |
| `clusterName`, `backendAddr`, `backendCA` | empty, `localhost:8443`, empty | Which cluster this is, the gateway to dial, the CA to trust for it |
| `enrollmentToken` or `enrollmentTokenSecret` | empty | The single-use token, or an existing Secret with key `token` |
| `observe`, `signalBuffer`, `debounce.*` | `true`, 1000, agent defaults | Watching and how long a failure must persist before a signal |
| `heartbeatInterval`, `logLevel`, `replicas` | `15s`, `info`, 1 | |
| `networkPolicy.enabled`, `networkPolicy.egressCIDRs` | `false`, empty | Optional policy: no ingress, egress to DNS and the listed CIDRs (list the gateway and the Kubernetes API server) |
| `podDisruptionBudget` | enabled, `maxUnavailable: 1` | |
| `resources` | 25m / 128Mi requested, 250m / 512Mi limit | Memory is what the cache needs; see Resources |
| `probes.enabled`, `probes.liveness`, `probes.readiness`, `statusDir` | on, see `values.yaml`, `/var/run/kubewatcher` | Exec probes (`cluster-agent healthcheck --live` / `--ready`) that read status files the agent keeps in an `emptyDir`; no port is opened. Liveness fails only if the agent's main loop made no progress for 5 minutes, never because the backend is unreachable |
| `goMemLimit` | `400MiB` | Becomes `GOMEMLIMIT`, a soft limit for the Go runtime, about 80% of the memory limit. Empty leaves it unset. Keep it below `resources.limits.memory` |

## Uninstall

```bash
helm uninstall kubewatcher-agent -n kubewatcher-system
kubectl -n kubewatcher-system delete secret kubewatcher-agent-identity   # created by the agent, not by Helm
```

Revoke the cluster in the dashboard as well.

## Upgrading

Agents live in customer clusters and cannot be upgraded in lockstep with the backend, so the backend accepts the current
and the previous protocol version. Upgrade the agent within one protocol version of the backend; an agent older than that
is refused with a message telling you to upgrade.

```bash
helm upgrade kubewatcher-agent oci://registry-1.docker.io/kubewatcher/cluster-agent-chart --version <new> -n kubewatcher-system --reset-then-reuse-values
```

Use `--reset-then-reuse-values` (Helm 3.14 or newer, and Helm 4): it applies the new chart's defaults and keeps the settings you set yourself. Plain `--reuse-values` fails when the new chart adds a setting. With an older Helm, use `--reset-values` and repeat your own `--set` options.

* The identity Secret survives upgrades; the agent does not re-enroll. Upgrade while the agent is online: an agent enrolled by 0.1.0 has no refresh credential and gets one by renewing once at start (0.1.1 and later). The credential, kept in the Secret key `refresh_token`, lets an agent that was offline longer than its 24-hour certificate renew without a re-enroll (30 days idle at most).
* The agent restarts by itself when its saved token or the CA it trusts changes. The image is built for amd64 and arm64.
* Switching `mode` between `read-only` and `remediate` swaps the remediation role in the same upgrade and restarts the pod. Tasks
  already running are recovered safely: a redelivered task first checks whether its change already happened.
* Changing `remediation.bounds` takes effect on restart. The agent enforces them regardless of what the backend sends.
* If the backend rotates its grant signing key, agents learn the new key at their next connection; nothing to do.

## Resources

The agent caches the cluster's objects in memory, with the container environment, volumes and `managedFields` left out. The defaults (25m CPU / 128Mi requested, 250m / 512Mi limit, `GOMEMLIMIT` 400MiB) leave room for clusters with thousands of pods. For tens of thousands of pods raise `resources.limits.memory` and set `goMemLimit` to about 80% of it.
