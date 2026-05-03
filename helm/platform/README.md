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
| NATS `StatefulSet` + Services + ConfigMap | Optional — `Coordinator` JMAP backend for HA |
| `ExternalSecret` | Optional — sources Stalwart's admin/DB/S3/encryption credentials |
| `ServiceMonitor` | Optional — scrapes `/metrics` on the workload's HTTP port |
| Grafana dashboard | Optional — `ConfigMap` with the sidecar discovery label |
| `PrometheusRule` | Optional — availability + restart-loop alerts |

## Configuration

Full values in [`values.yaml`](values.yaml), schema in [`values.schema.json`](values.schema.json). Artifact Hub renders the schema as an interactive form.

The chart is split so that **workload-lifecycle** resources (StatefulSet, HPA, PDB, HTTPRoute, headless Service, PVC templates) live in [`stalwart`](../base/README.md), while **cross-cutting dependencies** (data/blob stores, ESO, observability) live here. This keeps each chart focused and lets operators opt out of platform wiring without losing the app.

## Wiring into the base chart

`helm install` prints a `NOTES.txt` block with the exact `values.yaml` snippet to paste into the base chart, parameterised with the right Secret names. Run `helm get notes <release>` to retrieve it later.

The pattern: the base chart's `config.json` references env-var names inline (`{"@type": "EnvironmentVariable", "variableName": "X"}`), and the base chart's `env:` array injects those vars from the Secrets this chart provisions (CNPG-managed PG creds, the Garage access/secret-key Secrets, etc.). No aggregation Secret is needed — Kubernetes does the composition natively.

### NATS coordinator (optional, HA only)

For HA deployments, Stalwart needs a `Coordinator` for cluster pub/sub. This chart can provision a minimal NATS server inline (not a subchart — matches the rest of the platform pattern):

```yaml
nats:
  enabled: true
  replicas: 3            # 1 for single-node, 3+ for clustered routes
  auth:
    enabled: true        # turn on for shared clusters
    token: changeme      # or set existingSecret to point at an externally-managed Secret
```

The base chart consumes the resulting endpoint via `STALWART_NATS_URL` and `STALWART_NATS_TOKEN` env vars (set `nats.enabled: true` over there too). The actual `Coordinator` JMAP object that ties them together is still applied via `stalwart-cli apply` — `NOTES.txt` prints a ready-to-paste fragment.

Alternative: if you already run Redis as Stalwart's in-memory store, set `Coordinator: Default` in your apply-plan instead and skip NATS entirely — Stalwart will reuse the Redis connection for coordination.

## Bootstrapping non-storage settings (Stalwart 0.16+)

In Stalwart 0.16+ only the data store lives on disk (`/etc/stalwart/config.json`). Blob/FTS/in-memory stores, listeners, OIDC providers, DKIM signatures, and accounts are JMAP objects inside the data store, applied with [`stalwart-cli apply`](https://stalw.art/docs/management/cli/apply) against the management endpoint.

This chart does not yet automate that step. The `NOTES.txt` block walks through the manual flow: enable `recoveryAdmin` on the base chart, port-forward the management port, run `stalwart-cli apply` with a declarative plan.

## Part of the Hiroba ecosystem

Scaffolded with [Hiroba](https://github.com/7K-Hiroba/Hiroba). Source and issues: <https://github.com/7K-Hiroba/stalwart>. Upstream: <https://github.com/stalwartlabs/stalwart>.
