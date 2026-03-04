# Project Guidelines

## Architecture

ArgoCD GitOps repo for the monitoring stack. Each application lives in `apps/<name>/` with two files:

| File | Purpose |
|---|---|
| `application.yaml` | ArgoCD `Application` CR — points to an external Helm chart at a pinned tag |
| `values.yaml` | Helm value overrides for the chart |

### Managed applications

| App | Chart repo | Chart path | Pinned revision |
|---|---|---|---|
| victoria-metrics-cluster | `VictoriaMetrics/helm-charts` | `charts/victoria-metrics-cluster` | `victoria-metrics-cluster-0.35.0` |
| victoria-logs-cluster | `VictoriaMetrics/helm-charts` | `charts/victoria-logs-cluster` | `victoria-logs-cluster-0.0.27` |

ArgoCD uses **multi-source** `$values` reference to pull `values.yaml` from this repo while sourcing the chart from the upstream Git repo.

## Project Conventions

- **One directory per app** under `apps/`. Directory name matches the ArgoCD Application name.
- **Pin chart versions** via `spec.source.targetRevision` in `application.yaml`. Never use `HEAD` or branch refs.
- **Ingress is not configured here** — ingress resources are managed in a separate ArgoCD application to allow independent lifecycle and shared ingress controller config.
- **Backups are not configured here** — vmbackup/vmrestore CronJobs are managed in a separate ArgoCD application (`vm-backup`) to isolate S3 credentials and allow independent scheduling.
- **Do not duplicate upstream defaults** in `values.yaml` — only override values that differ from the chart's defaults. Comment out optional overrides for reference.
- Retention, replication, and resource limits are the primary tuning knobs — they are explicitly present (even if commented) in each `values.yaml`.

## Key Configuration Patterns

### victoria-metrics-cluster

- **Retention**: `vmstorage.retentionPeriod` (default `"1"` = 1 month; supports `h/d/w/y` suffixes)
- **Replication**: set via `vminsert.extraArgs.replicationFactor`; must also enable dedup on vmselect with `vmselect.extraArgs.dedup.minScrapeInterval`
- **Components**: vmselect (query), vminsert (write), vmstorage (storage), vmauth (optional proxy)

### victoria-logs-cluster

- **Retention**: `vlstorage.retentionPeriod` (default `"7d"`); also supports `retentionDiskSpaceUsage` and `retentionMaxDiskUsagePercent`
- **Replication**: no explicit `replicationFactor` in v0.0.27 — distribution handled automatically by vlinsert across vlstorage replicas
- **Components**: vlselect (query), vlinsert (write), vlstorage (storage), vmauth (optional proxy), vector (optional log collector)

## Build and Test

No build step. Validate changes with:

```bash
# Lint ArgoCD Application manifests
kubectl apply --dry-run=client -f apps/victoria-metrics-cluster/application.yaml
kubectl apply --dry-run=client -f apps/victoria-logs-cluster/application.yaml

# Validate YAML syntax
yamllint apps/
```

## Security

- No secrets should be committed to this repo. Use ArgoCD's external secret management or SealedSecrets.
- `vmauth` component can be enabled to add authentication in front of read/write endpoints.
