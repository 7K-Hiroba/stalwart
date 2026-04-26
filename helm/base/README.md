# stalwart

Helm chart for [Stalwart](https://stalw.art/) — an all-in-one mail and collaboration server (SMTP, IMAP, POP3, JMAP, CalDAV, CardDAV, WebDAV, ManageSieve).

This chart deploys the workload itself: a Deployment, multi-port Service, Gateway API HTTPRoute for HTTP-based protocols, and PVCs for `/etc/stalwart` and `/var/lib/stalwart`. It uses the [official upstream image](https://github.com/stalwartlabs/stalwart/pkgs/container/stalwart) — no custom Dockerfile.

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

- Kubernetes 1.24+
- Gateway API CRDs installed in the cluster (the chart provisions an `HTTPRoute`)
- A running `Gateway` that the `HTTPRoute` can attach to (HTTP-based protocols only)
- A way to expose mail TCP ports (`LoadBalancer`, MetalLB, NodePort + edge proxy) if internet mail is in scope
- A `StorageClass` for the config and data PVCs

## What gets installed

| Resource | Purpose |
|---|---|
| `Deployment` | Single-instance Stalwart pod, runs as UID 2000 |
| `Service` | Multi-port (HTTP + every mail protocol), defaults to `ClusterIP` |
| `HTTPRoute` | Gateway API routing for the HTTP port (admin UI, JMAP, *DAV) |
| `PersistentVolumeClaim` × 2 | `config` (1 Gi) and `data` (20 Gi), both `ReadWriteOnce` |
| `ServiceAccount` | Pod identity, no token automount |
| `HorizontalPodAutoscaler` | Optional — disabled by default (state in PVCs) |
| `PodDisruptionBudget` | Optional — useful only with multi-replica setups |

## Configuration

All values are documented in [`values.yaml`](values.yaml) and validated against [`values.schema.json`](values.schema.json). Artifact Hub renders the schema as an interactive form on the chart page.

Stalwart bootstraps via the in-cluster admin UI on first run — port-forward `svc/stalwart 8080:8080` after install and follow the wizard. The generated admin password is printed to the pod logs.

## Part of the Hiroba ecosystem

Scaffolded with [Hiroba](https://github.com/7K-Hiroba/Hiroba). Source and issues: <https://github.com/7K-Hiroba/stalwart>. Upstream: <https://github.com/stalwartlabs/stalwart>.
