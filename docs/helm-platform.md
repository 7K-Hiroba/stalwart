---
sidebar_position: 4
---

# Helm platform chart

The platform chart provisions resources that **surround** Stalwart: a PostgreSQL data store, an S3-compatible blob store, ExternalSecrets for credentials, and observability (ServiceMonitor, Grafana dashboard, PrometheusRules). It is optional — install the [base chart](./helm-base.md) alone for a minimal single-instance deployment with embedded RocksDB + on-disk blob store.

Every section ships **disabled by default** so the chart is safe to install into clusters that don't have the relevant operators present. Enabling a feature without its required CRDs causes `helm install` to fail early with a clear error rather than silently creating Custom Resources nothing is reconciling.

## Matching the base chart

The ServiceMonitor below needs to select the base chart's pods. Set `global.baseInstance` to the base chart's release name so the selector matches both `app.kubernetes.io/name` and `app.kubernetes.io/instance`:

```yaml
global:
  baseInstance: stalwart   # release name used for `helm install <name> ./helm/base`
```

Leave it empty to match by name only — fine when only one Stalwart release runs in the cluster.

> PodDisruptionBudget lives in the [base chart](./helm-base.md#poddisruptionbudget), not here — it's tightly coupled to the Deployment's lifecycle.

Values reference: [`helm/platform/values.yaml`](https://github.com/7K-Hiroba/stalwart/blob/main/helm/platform/values.yaml)

## Install

```bash
helm install stalwart-platform ./helm/platform \
  --set postgres.enabled=true \
  --set externalSecrets.enabled=true \
  --set observability.serviceMonitor.enabled=true
```

## PostgreSQL data store

Provisions a PostgreSQL cluster via [CloudNativePG](https://cloudnative-pg.io/). Stalwart uses it as the **data store backend** — account metadata, mailbox state, ACLs, sessions. Without it Stalwart falls back to RocksDB on the data PVC, which is fine for small homelab installs but doesn't scale to multiple replicas.

### Prerequisites

- CloudNativePG operator installed in the cluster
- A `StorageClass` available for the data volume

### Configuration

```yaml
postgres:
  enabled: true
  provider: cnpg
  instances: 1            # 3 for HA
  storage:
    size: 10Gi
    storageClass: ""      # cluster default
  database: stalwart
  owner: stalwart
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retentionPolicy: "7d"
```

CNPG publishes connection credentials to a `Secret` named `stalwart-pg-app`. Wire it into Stalwart by either:

- adding `envFrom: [{ secretRef: { name: stalwart-pg-app } }]` to the base chart values, then referencing the env vars in `config.json`
- mapping the keys via ExternalSecrets (below) into a single combined Secret

## S3 blob store

Provisions an S3-compatible bucket. Stalwart uses it as the **blob store backend** — raw message bytes, attachments, Sieve scripts. Two providers:

- **`crossplane`** — provisions a real bucket on AWS (or an S3-compatible cloud) via Crossplane's S3 provider
- **`garage`** — creates a bucket on an in-cluster [Garage](https://garagehq.deuxfleurs.fr/) deployment (homelab-friendly)

### Configuration

```yaml
s3:
  enabled: true
  provider: garage
  bucketName: blob
  acl: private
  garage:
    endpoint: "http://garage.garage.svc.cluster.local:3900"
    accessKeySecret: { name: garage-stalwart, key: accessKey }
    secretKeySecret: { name: garage-stalwart, key: secretKey }
    replicationFactor: 3
```

Switch backends by changing `s3.provider` — the provider-specific blocks (`crossplane`, `garage`) configure each implementation.

You'll still need to point Stalwart's blob store at the bucket. The simplest path is to put the access key, secret key, and endpoint into the ExternalSecret below, then reference them from `config.json` via env-var interpolation.

## ExternalSecrets

Populates a Kubernetes `Secret` from an upstream store (Vault, AWS Secrets Manager, 1Password, etc.) via an `ExternalSecret` resource. The base chart's `envFrom` then pulls credentials from this `Secret`.

### Prerequisites

- [external-secrets operator](https://external-secrets.io/) installed in the cluster
- A `ClusterSecretStore` (or `SecretStore`) configured and reachable

### Configuration

```yaml
externalSecrets:
  enabled: true
  refreshInterval: 1h
  storeRef:
    name: cluster-secret-store
    kind: ClusterSecretStore
  data:
    - secretKey: STALWART_ADMIN_PASSWORD
      remoteKey: stalwart/admin
      property: password
    - secretKey: STALWART_STORE_DB_PASSWORD
      remoteKey: stalwart/postgres
      property: password
    - secretKey: STALWART_STORE_BLOB_ACCESS_KEY
      remoteKey: stalwart/s3
      property: accessKey
    - secretKey: STALWART_STORE_BLOB_SECRET_KEY
      remoteKey: stalwart/s3
      property: secretKey
```

To pull every key under a remote path instead of mapping them individually, use `dataFrom`:

```yaml
externalSecrets:
  dataFrom:
    - extract:
        key: stalwart/config
```

### Value transformation

Use `target.template` to synthesize new secret values from the retrieved ones (e.g. compose a connection string from parts). See the [ESO templating guide](https://external-secrets.io/latest/guides/templating/):

```yaml
externalSecrets:
  target:
    template:
      type: Opaque
      data:
        STALWART_STORE_DB_URL: "postgres://{{ `{{ .username }}` }}:{{ `{{ .password }}` }}@host/stalwart"
```

The double-brace escape (`{{ ` ... ` }}`) is needed because Helm processes the values file first; the inner braces reach ESO untouched.

### Wiring back into the base chart

The generated `Secret` is named after the application (`stalwart`). Reference it from the base chart's `envFrom`:

```yaml
# helm/base values override
envFrom:
  - secretRef:
      name: stalwart
```

Stalwart's [config supports environment-variable interpolation](https://stalw.art/docs/configuration/overview), so `STALWART_*` env vars can be read directly from `config.json` (e.g. `"value": "%{env:STALWART_STORE_DB_PASSWORD}%"`).

## Observability

### ServiceMonitor

Scrapes Stalwart's `/metrics` endpoint via the Prometheus Operator. Metrics are served on the same port as the admin UI (`http`, port 8080).

```yaml
observability:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: kube-prometheus-stack  # match your Prometheus instance selector
    port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

Requires the Prometheus Operator CRDs (`monitoring.coreos.com/v1`).

### Grafana dashboard

Deploys dashboards as a `ConfigMap` with the Grafana sidecar label. The sidecar picks them up and imports them into Grafana automatically.

```yaml
observability:
  grafanaDashboard:
    enabled: true
    folderLabel: "Mail"
```

Dashboards are loaded from `helm/platform/dashboards/*.json`. The shipped dashboard is a placeholder using generic `http_*` metrics — replace it with panels driven by Stalwart's actual metric names (`stalwart_smtp_*`, `stalwart_imap_*`, queue depth, delivery latency) once you've confirmed which ones your build exposes.

### PrometheusRules

Ships availability-style alerts that don't depend on Stalwart-specific metric names — `up == 0` (scrape failed) and container restart loops. Enable, then layer protocol-specific alerts on top once you've inspected `/metrics` on a running instance.

```yaml
observability:
  prometheusRules:
    enabled: true
    groups:
      - name: stalwart.protocols
        rules:
          - alert: StalwartSMTPHighRejectRate
            expr: |
              rate(stalwart_smtp_rejected_total[5m]) > 5
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "Stalwart is rejecting SMTP at >5/s"
```

Override the whole list to replace the built-in alerts, or append to extend them. Requires the Prometheus Operator CRDs.

## What this chart does NOT install

- The parent `Gateway` resource — provided by a gateway chart (HTTP only; mail TCP doesn't go through the Gateway)
- TLS certificates for the Gateway listener — expected to be wired into the Gateway
- LoadBalancer / NodePort exposure for mail TCP — that's a base-chart `service.type` decision
- Cluster-wide operators (CNPG, Crossplane, External Secrets, Prometheus Operator) — these are platform prerequisites
