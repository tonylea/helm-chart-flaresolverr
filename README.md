# FlareSolverr Helm Chart

![Downloads](https://img.shields.io/github/downloads/tonylea/helm-chart-flaresolverr/total)

Deploys [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) from its official image, [`ghcr.io/flaresolverr/flaresolverr`](https://github.com/FlareSolverr/FlareSolverr/pkgs/container/flaresolverr).
FlareSolverr is a proxy server that drives a headless Chromium to solve Cloudflare and DDoS-GUARD challenges, so other apps, such as indexer managers, can reach the sites behind them.

The chart runs one replica:

- A Deployment that stops the old pod before starting the new one.
  FlareSolverr keeps no state, so it has no volume besides `/tmp`.
- A Service on the API port, 8191, and the metrics port, 8192.
  FlareSolverr has no web UI and its API has no authentication, so the chart has no `HTTPRoute`; reach it from inside the cluster.
- A start-up, liveness and readiness probe on `/health`.
  It only shows the process is up, since FlareSolverr launches Chromium once at start and exits if it can't.
- An optional ServiceMonitor on the metrics port.

The container runs as the image's own non-root user, with all capabilities dropped and no privilege escalation.
The chart sets no `runAsUser`, because the image leaves the UID unset, and no `readOnlyRootFilesystem`, because Chromium writes under `/app`.

## Values

| Value | Default |
| --- | --- |
| `image.tag` | the chart's `appVersion` |
| `resources` | `{}`; no requests or limits |
| `affinity` | none |
| `serviceMonitor.enabled` | `true` |
| `serviceMonitor.interval` | `30s` |

## Fixed settings

- `PROMETHEUS_ENABLED` is `true`, so the metrics port serves request counts and durations by domain.
- `/tmp` is an in-memory `emptyDir` of 256Mi, the size the project's README recommends for Chromium's per-request files.

## Requirements

- The Prometheus Operator's CRDs when `serviceMonitor.enabled` is set.

## Usage

```sh
helm repo add flaresolverr https://tonylea.github.io/helm-chart-flaresolverr
helm install flaresolverr flaresolverr/flaresolverr --values my-values.yaml
```

Other apps in the cluster then use `http://flaresolverr:8191` as the FlareSolverr URL, from the same namespace.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 512Mi
```

## Development

Pull requests run:

- `helm lint --strict`, and the rendered chart checked by kubeconform against Kubernetes 1.37.
- helm-unittest's tests in `tests/`.
- An install with chart-testing in a kind cluster, with the ServiceMonitor off because kind has no Prometheus Operator CRDs.

Commits follow Conventional Commits.
Releases are versioned and tagged automatically from `main`, and published to the Helm repository on this repo's GitHub Pages.
