# Changelog

Releases of the KubeWatcher agent chart, newest first. The same notes are on each GitHub release.

## 0.2.1 - 2026-11-22

The agent image and chart now live in the kubewatcher organization on Docker Hub; the chart is renamed cluster-agent-chart. The agent itself is unchanged.

Full diff: https://github.com/kubewatcher/charts/compare/v0.2.0...v0.2.1

## 0.2.0 - 2026-11-21

Groups incidents by what changed: reports a pod template fingerprint and detects hand-made workload changes. Bounded signal buffer, health probes, lower memory use.

Full diff: https://github.com/kubewatcher/charts/compare/v0.1.1...v0.2.0

## 0.1.1 - 2026-11-15

First public release of the agent chart.
