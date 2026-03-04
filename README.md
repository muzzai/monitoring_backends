# monitoring_stack

ArgoCD GitOps repo for monitoring backends. Uses **App of Apps** pattern — deploy only the root application, everything else syncs automatically.

## Quick start

```bash
kubectl apply -f apps/root/application.yaml
```

## Structure

```
apps/
├── root/                          # App of Apps — scans apps/*/application.yaml
│   └── application.yaml
├── victoria-metrics-cluster/      # Time-series metrics (VictoriaMetrics cluster)
│   ├── application.yaml           # Chart pinned at victoria-metrics-cluster-0.35.0
│   └── values.yaml                # Overrides: retention, replication, resources
├── victoria-logs-cluster/         # Log storage (VictoriaLogs cluster)
│   ├── application.yaml           # Chart pinned at victoria-logs-cluster-0.0.27
│   └── values.yaml                # Overrides: retention, disk limits, resources
└── grafana/                       # Dashboards & visualization
    ├── application.yaml           # Chart pinned at grafana-11.2.3
    └── values.yaml                # Overrides: datasources, plugins, persistence
```

## Design decisions

- **Ingress** — managed in a separate ArgoCD application (independent lifecycle, shared controller).
- **Backups** — managed in a separate ArgoCD application (`vm-backup`); S3 credentials isolated.
- **Secrets** — never committed; use SealedSecrets or ArgoCD external secret management.
- **values.yaml** — only overrides, no upstream defaults duplicated. Optional settings commented out for reference.
- **Chart versions** — pinned via `targetRevision` in `application.yaml`. Never `HEAD`.

## Validation

```bash
kubectl apply --dry-run=client -f apps/victoria-metrics-cluster/application.yaml
kubectl apply --dry-run=client -f apps/victoria-logs-cluster/application.yaml
kubectl apply --dry-run=client -f apps/grafana/application.yaml
yamllint apps/
```

## AI agent notes

Full workspace conventions are in `.github/copilot-instructions.md`. Key points:
- One directory per app under `apps/`. Directory name = ArgoCD Application name.
- Charts are sourced from `github.com/VictoriaMetrics/helm-charts.git` via multi-source `$values` reference.
- Primary tuning knobs: retention (`retentionPeriod`), replication (`extraArgs.replicationFactor` for VM only), resource limits.
- VictoriaLogs v0.0.27 has no explicit replication factor — vlinsert distributes writes automatically.
