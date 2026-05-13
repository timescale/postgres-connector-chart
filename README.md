# postgres-connector

Helm chart that deploys the Timescale Postgres connector — a stateless service that replicates Postgres into TimescaleDB. Runs on any Kubernetes flavor, including OpenShift.

## Install

Pick a release from the [Releases page](https://github.com/timescale/postgres-connector-chart/releases) and install the packaged chart directly:

```sh
helm install pgc \
  https://github.com/timescale/postgres-connector-chart/releases/download/v0.1.0/postgres-connector-0.1.0.tgz \
  -f my-values.yaml
```

Or clone and install from source:

```sh
helm install pgc . -f my-values.yaml
```

## Required values

```yaml
config:
  version: 1
  connectors:
    - id: my-connector
      source:
        database_url: $SOURCE_URL
        publications:
          - my_publication
      target:
        database_url: $TARGET_URL
        tables:
          - source:
              schema_name: public
              table_name: metrics
            hypertable_config:
              primary_dimension:
                column_name: time
                range:
                  partition_interval: 1 day

env:
  - name: SOURCE_URL
    valueFrom:
      secretKeyRef: { name: db-creds, key: source-url }
  - name: TARGET_URL
    valueFrom:
      secretKeyRef: { name: db-creds, key: target-url }
```

The `config` block is rendered into a Kubernetes Secret and mounted at `/etc/connector/config.yaml`. Use `$VAR` expansion for credentials and inject the variables via `env`.

The full config schema lives in [`live-sync/pkg/config/config.go`](https://github.com/timescale/live-sync/blob/main/pkg/config/config.go).

## Updating the config

Edit `config:` in your values file and run `helm upgrade`. The connector polls the mounted file every 30 seconds and reconciles connectors without a pod restart.

## Monitoring

- **Prometheus**: a `ServiceMonitor` is created by default (requires the Prometheus Operator). Disable with `serviceMonitor.enabled: false`.
- **Grafana**: a `ConfigMap` labeled `grafana_dashboard: "1"` is created for the kube-prometheus-stack Grafana sidecar. Disable with `grafana.dashboard.enabled: false`.

## OpenShift

Works under the default `restricted` SCC without modification — the chart sets no `runAsUser` and ships only non-root, capability-dropped containers with a read-only root filesystem.

## Values

See [`values.yaml`](values.yaml).

## Releases

Releases are automated by [release-please](https://github.com/googleapis/release-please). Use [Conventional Commits](https://www.conventionalcommits.org/) — `feat:` bumps the minor version, `fix:` bumps the patch, `feat!:` / `BREAKING CHANGE:` bumps the major. release-please opens a release PR on every push to `main`; merging it creates a tag, a GitHub Release, and bumps `version` in `Chart.yaml`.
