# postgres-connector

Helm chart that deploys the Timescale Postgres connector — a stateless service that replicates Postgres into TimescaleDB. Runs on any Kubernetes flavor, including OpenShift.

## Install

Pick a release from the [Releases page](https://github.com/timescale/postgres-connector-chart/releases) and install the packaged chart directly:

```sh
helm install postgres-connector \
  https://github.com/timescale/postgres-connector-chart/releases/download/v0.1.0/postgres-connector-0.1.0.tgz \
  -f my-values.yaml
```

To always install the latest release, resolve the tag at install time (the chart `.tgz` is a versioned asset, so there is no fixed "latest" URL):

```sh
REPO=timescale/postgres-connector-chart
TAG=$(gh release view --repo "$REPO" --json tagName -q .tagName)
VERSION=${TAG#v}
helm install postgres-connector \
  "https://github.com/$REPO/releases/download/$TAG/postgres-connector-$VERSION.tgz" \
  -f my-values.yaml
```

Or clone and install from source (use this to track `main` ahead of a release):

```sh
helm install postgres-connector . -f my-values.yaml
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

The `config` block is rendered into a Kubernetes Secret and mounted at `/etc/connector/config.yaml`. Use `$VAR` expansion in `database_url` fields for credentials and inject the variables via `env`.

## Config schema

Top-level:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `version` | int | yes | Config schema version. Must be `1`. |
| `connectors` | list | yes | One or more connector definitions. At least one is required. |

### `connectors[]`

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | yes | Unique connector identifier. Either a UUID or a string of alphanumerics, underscores, and hyphens. Must be unique across the file. |
| `drop` | bool | no | When `true`, the connector is torn down (source-side replication slot/publication state and target cleanup) instead of running. |
| `config` | object | no | Replication tuning (see below). Defaults applied when omitted. |
| `source` | object | yes | Source Postgres configuration. |
| `target` | object | yes | Target TimescaleDB configuration. |

### `connectors[].config` (replication)

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | bool | `true` | Whether replication is active for this connector. |
| `table_sync_workers` | int | `4` | Parallel workers used during the initial table sync. Must be 1–100. |
| `initial_data_copy` | bool | `true` | Whether to perform the initial snapshot copy before streaming changes. Set to `false` to skip the snapshot and stream from the current LSN. |

### `connectors[].source`

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `database_url` | string | yes | Postgres connection string for the source. Supports `$VAR` expansion from environment. |
| `publications` | list of strings | yes | Existing Postgres logical-replication publications to subscribe to. At least one is required. |
| `ssh_config` | object | no | SSH tunnel for reaching the source (see below). |

### `connectors[].source.ssh_config`

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `host` | string | yes | SSH bastion host. |
| `port` | int | no (default `22`) | SSH port (1–65535). |
| `user` | string | yes | SSH user. |
| `password` | string | no | SSH password (use private key auth where possible). |
| `private_key` | string | yes | Base64-encoded private key, or `@/path/to/keyfile` to load and encode from disk. |
| `host_key` | string | no | Expected host public key for verification. |

### `connectors[].target`

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `database_url` | string | yes | TimescaleDB/Postgres connection string for the target. Supports `$VAR` expansion. |
| `skip_replica_identity_index` | bool | no | Global default for the per-table setting below. |
| `tables` | list | no | Per-table source-to-target mapping and hypertable configuration. |

### `connectors[].target.tables[]`

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `source` | object | yes | Source table: `{ schema_name, table_name }`. |
| `target` | object | no | Target table: `{ schema_name, table_name }`. Defaults to the source pair when omitted. |
| `hypertable_config` | object | no | If set, creates a hypertable on the target with the given dimensions. |
| `skip_replica_identity_index` | bool | no | When `true`, skip creating the replica identity index on the target. Falls back to the target-level default. |

### `connectors[].target.tables[].hypertable_config`

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `primary_dimension` | object | yes | Primary partitioning dimension. Must be a `range` dimension. |
| `secondary_dimensions` | list | no | Additional dimensions, each either `range` or `hash`. |

Each dimension has:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `column_name` | string | yes | Column used for partitioning. |
| `range.partition_interval` | string | one of | Time/interval per chunk, e.g. `1 day`. |
| `hash.num_partitions` | int | one of | Number of hash partitions. Must be ≥ 2. Not allowed on `primary_dimension`. |

Exactly one of `range` or `hash` must be set per dimension.

## Updating the config

Edit `config:` in your values file and run `helm upgrade`. The connector polls the mounted file every 30 seconds and reconciles connectors without a pod restart.

## Monitoring progress

The connector exposes its internal state in the target database under the `_ts_live_sync` schema and uses logical replication slots named `live_sync_*` on the source. Set `SOURCE` and `TARGET` to the same connection strings used in your config.

### Table sync status

```sql
psql "$TARGET" -c "
  SELECT state, last_error, count(*)
  FROM _ts_live_sync.subscription_rel
  GROUP BY 1, 2 ORDER BY 1, 2;
"
```

Tables with errors appear as separate rows grouped by `last_error`.

| State | Meaning |
| --- | --- |
| `i` | Initial state, table data sync not started |
| `d` | Initial table data sync is in progress |
| `f` | Initial table data sync completed, catching up with incremental changes |
| `s` | Synchronized, waiting for the main apply worker to take over |
| `r` | Table is ready, applying changes in real-time |

### COPY progress during initial sync

The number of rows returned should usually match `table_sync_workers` from the connector config.

```sql
psql "$SOURCE" -c "SELECT * FROM pg_stat_progress_copy;"
```

### Replication lag

On the source, via `pg_replication_slots`:

```sql
psql "$SOURCE" -c "
  SELECT slot_name,
    pg_size_pretty(pg_current_wal_flush_lsn() - confirmed_flush_lsn) AS lag
  FROM pg_replication_slots
  WHERE slot_name LIKE 'live_sync_%' AND slot_type = 'logical';
"
```

On the target, via `_ts_live_sync.subscription` (also surfaces `last_error`):

```sql
psql "$TARGET" -c "
  SELECT
    pg_size_pretty(source_flush_lsn - last_replicated_lsn) AS lag_bytes,
    (metrics_updated_at - last_replicated_txn_time) AS lag_duration,
    *
  FROM _ts_live_sync.subscription;
"
```

## Metrics and dashboards

- **Prometheus**: a `ServiceMonitor` is created by default (requires the Prometheus Operator). Disable with `serviceMonitor.enabled: false`.
- **Grafana**: a `ConfigMap` labeled `grafana_dashboard: "1"` is created for the kube-prometheus-stack Grafana sidecar. Disable with `grafana.dashboard.enabled: false`.

## OpenShift

Works under the default `restricted` SCC without modification — the chart sets no `runAsUser` and ships only non-root, capability-dropped containers with a read-only root filesystem.

## Values

See [`values.yaml`](values.yaml).

## Releases

Releases are automated by [release-please](https://github.com/googleapis/release-please). Use [Conventional Commits](https://www.conventionalcommits.org/) — `feat:` bumps the minor version, `fix:` bumps the patch, `feat!:` / `BREAKING CHANGE:` bumps the major. release-please opens a release PR on every push to `main`; merging it creates a tag, a GitHub Release, and bumps `version` in `Chart.yaml`.
