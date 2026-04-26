# stalwart-platform

Platform dependencies for the [stalwart](../base/README.md) chart — provisions the cross-cutting resources Stalwart needs but that aren't part of its workload lifecycle: PostgreSQL data store, S3 blob store, ExternalSecrets for credentials, observability (ServiceMonitor, Grafana dashboard, PrometheusRules).

Install **alongside** the `stalwart` chart, typically in the same namespace.

**Documentation:** <https://hiroba.7kgroup.org/docs/apps/stalwart/helm-platform>

## TL;DR

```bash
helm install stalwart-platform \
  oci://harbor.7kgroup.org/7khiroba/charts/stalwart-platform \
  --version 0.1.0 \
  --set postgres.enabled=true \
  --set externalSecrets.enabled=true
```

## Prerequisites

Operators are required only for the features you enable:

- [CloudNativePG](https://cloudnative-pg.io/) — for the `postgres` data store
- [Crossplane](https://www.crossplane.io/) (with AWS S3 provider) or [Garage](https://garagehq.deuxfleurs.fr/) — for the `s3` blob store
- [External Secrets Operator](https://external-secrets.io/) — for the `ExternalSecret`
- [Prometheus Operator](https://prometheus-operator.dev/) — for `ServiceMonitor` / `PrometheusRule`
- Grafana with dashboard sidecar enabled — to pick up shipped dashboards

The chart fails fast at template-render time if you enable a feature whose CRDs aren't installed, rather than silently creating orphan Custom Resources.

## What gets installed

| Resource | Purpose |
|---|---|
| `Cluster` (CNPG) | Optional — PostgreSQL data store backend |
| Crossplane `Bucket` / Garage `ConfigMap` + Job | Optional — S3 blob store backend |
| `ExternalSecret` | Optional — sources Stalwart's admin/DB/S3/encryption credentials |
| `ServiceMonitor` | Optional — scrapes `/metrics` on the workload's HTTP port |
| Grafana dashboard | Optional — `ConfigMap` with the sidecar discovery label |
| `PrometheusRule` | Optional — availability + restart-loop alerts |

## Configuration

Full values in [`values.yaml`](values.yaml), schema in [`values.schema.json`](values.schema.json). Artifact Hub renders the schema as an interactive form.

The chart is split so that **workload-lifecycle** resources (Deployment, HPA, PDB, HTTPRoute, PVCs) live in [`stalwart`](../base/README.md), while **cross-cutting dependencies** (data/blob stores, ESO, observability) live here. This keeps each chart focused and lets operators opt out of platform wiring without losing the app.

## Part of the Hiroba ecosystem

Scaffolded with [Hiroba](https://github.com/7K-Hiroba/Hiroba). Source and issues: <https://github.com/7K-Hiroba/stalwart>. Upstream: <https://github.com/stalwartlabs/stalwart>.
