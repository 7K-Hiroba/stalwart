# stalwart

Helm chart for [Stalwart](https://stalw.art/) — an all-in-one mail and collaboration server (SMTP, IMAP, POP3, JMAP, CalDAV, CardDAV, WebDAV, ManageSieve).

This chart deploys the workload itself: a StatefulSet, multi-port Service, headless Service for stable per-pod DNS, Gateway API HTTPRoute for HTTP-based protocols, a Secret holding `/etc/stalwart/config.json`, and a `volumeClaimTemplates`-managed PVC for `/var/lib/stalwart`. It uses the [official upstream image](https://github.com/stalwartlabs/stalwart/pkgs/container/stalwart) — no custom Dockerfile.

For cross-cutting platform dependencies (PostgreSQL data store, S3 blob store, secrets, observability), install the companion [stalwart-platform](../platform/README.md) chart.

**Documentation:** <https://hiroba.7kgroup.org/docs/apps/stalwart/helm-base>

## TL;DR

```bash
helm install stalwart \
  oci://harbor.7kgroup.org/7khiroba/charts/stalwart \
  --version 0.1.0 \
  --set gateway.hostnames[0]=mail.example.com
```

## Prerequisites

- Kubernetes 1.27+ (`StatefulSet.spec.ordinals` is alpha in 1.26, beta in 1.27, stable in 1.31)
- Gateway API CRDs installed in the cluster (the chart provisions an `HTTPRoute`)
- A running `Gateway` that the `HTTPRoute` can attach to (HTTP-based protocols only)
- A way to expose mail TCP ports (`LoadBalancer`, MetalLB, NodePort + edge proxy) if internet mail is in scope
- A `StorageClass` for the data PVC

## What gets installed

| Resource | Purpose |
|---|---|
| `StatefulSet` | Stalwart pods with stable identity (1-based ordinals), `volumeClaimTemplates` for the data PVC |
| `Service` (ClusterIP) | Multi-port load-balancing service (HTTP + every mail protocol) |
| `Service` (headless) | Per-pod DNS for cluster identity (`<pod>.<fullname>-headless`) |
| `Secret` (`-config`) | Holds `/etc/stalwart/config.json` rendered from `values.config` |
| `Secret` (`-env`) | Optional — self-managed env Secret when `secrets.create: true` |
| `HTTPRoute` | Gateway API routing for the HTTP port (admin UI, JMAP, *DAV) |
| `ServiceAccount` | Pod identity, no token automount |
| `HorizontalPodAutoscaler` | Optional — disabled by default (state in PVCs) |
| `PodDisruptionBudget` | Optional — useful only with multi-replica setups |

## Configuration

All values are documented in [`values.yaml`](values.yaml) and validated against [`values.schema.json`](values.schema.json). Artifact Hub renders the schema as an interactive form on the chart page.

### config.json

Stalwart 0.16+ keeps **only the data store** in `/etc/stalwart/config.json`; everything else (blob/FTS/in-memory backends, listeners, OIDC, DKIM, accounts) lives inside the data store as JMAP objects, applied with [`stalwart-cli apply`](https://stalw.art/docs/management/cli/apply).

The chart renders `config.json` from `values.config`. Default is embedded RocksDB at `/var/lib/stalwart`. To point at PostgreSQL with secrets injected via env vars:

```yaml
config:
  "@type": Postgres
  host: stalwart-pg-rw.stalwart.svc.cluster.local
  port: 5432
  database: stalwart
  user: stalwart
  password:
    "@type": EnvironmentVariable
    variableName: STALWART_DATA_PASSWORD

env:
  - name: STALWART_DATA_PASSWORD
    valueFrom:
      secretKeyRef:
        name: stalwart-pg-app
        key: password
```

The companion platform chart prints a ready-to-paste version of this block in its install `NOTES.txt`.

### Bootstrapping the rest (NOT yet automated)

This chart does not yet automate `stalwart-cli apply`. On first install, do it manually:

```bash
# 1. Set on base chart values
recoveryAdmin:
  enabled: true
secrets:
  create: true
  env:
    STALWART_RECOVERY_ADMIN: "admin:tempPassword"

# 2. After helm install, port-forward and apply your plan:
kubectl -n stalwart port-forward svc/stalwart 8080:8080
STALWART_URL=http://127.0.0.1:8080 \
STALWART_USER=admin \
STALWART_PASSWORD=tempPassword \
stalwart-cli apply --file plan.json

# 3. After provisioning a real admin Principal in the plan, disable recoveryAdmin and helm upgrade.
```

## Part of the Hiroba ecosystem

Scaffolded with [Hiroba](https://github.com/7K-Hiroba/Hiroba). Source and issues: <https://github.com/7K-Hiroba/stalwart>. Upstream: <https://github.com/stalwartlabs/stalwart>.
