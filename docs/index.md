---
sidebar_position: 1
---

# Stalwart

[Stalwart](https://stalw.art/) is an open-source mail and collaboration server written in Rust. A single binary handles SMTP, IMAP4, POP3, JMAP, ManageSieve, CalDAV, CardDAV, and WebDAV — replacing the typical Postfix + Dovecot + Radicale stack with one process.

This Hiroba application packages Stalwart for Kubernetes following the [near-native philosophy](https://hiroba.7kgroup.org/docs/architecture/near-native): we ship the [official upstream image](https://github.com/stalwartlabs/stalwart/pkgs/container/stalwart) and a thin Helm chart around it. There is no custom Dockerfile.

## Architecture

The chart is split into two pieces with different lifecycles:

- **`helm/base`** — the workload itself. Deployment, multi-port Service (HTTP + every mail protocol), Gateway API HTTPRoute for the admin UI/JMAP, and PVCs for `/etc/stalwart` and `/var/lib/stalwart`.
- **`helm/platform`** — optional cross-cutting dependencies: PostgreSQL data store via CloudNativePG, S3 blob store via Crossplane or Garage, ExternalSecrets for credentials, ServiceMonitor + Grafana dashboard + PrometheusRules for observability.

Stalwart can run standalone (RocksDB on the data PVC) for a homelab single-tenant install, or with PostgreSQL + S3 once you outgrow that. The platform chart's `enabled` flags let you turn each piece on independently.

## Prerequisites

- Kubernetes 1.24+ with the [Gateway API](https://gateway-api.sigs.k8s.io/) CRDs installed
- A parent `Gateway` (HTTP + HTTPS listeners with TLS) the HTTPRoute can attach to — typically from a gateway chart such as `hiroba-gateway`
- A way to expose mail TCP ports (LoadBalancer, MetalLB, or NodePort + external proxy) if you intend to receive internet mail
- Helm v3.x and `kubectl`
- Operators are required only for the platform features you enable (CNPG, Crossplane, External Secrets Operator, Prometheus Operator)

## Quick start

```bash
# Workload — single-instance Stalwart with embedded data store
helm install stalwart ./helm/base \
  --set gateway.hostnames[0]=mail.example.com

# Platform dependencies — opt in per category
helm install stalwart-platform ./helm/platform \
  --set postgres.enabled=true \
  --set externalSecrets.enabled=true \
  --set observability.serviceMonitor.enabled=true
```

After install, port-forward the admin UI to bootstrap:

```bash
kubectl port-forward svc/stalwart 8080:8080
# → open http://localhost:8080 and follow the setup wizard
```

The wizard prints a generated admin password on first run — capture it from the pod logs (`kubectl logs deploy/stalwart`) before restarting.

## Per-stack documentation

- **[Container image](./container.md)** — upstream image, pinning, what we don't customize
- **[Helm base chart](./helm-base.md)** — Deployment, Service, HTTPRoute, mail port exposure, persistence
- **[Helm platform chart](./helm-platform.md)** — PostgreSQL, S3 blob store, ExternalSecrets, observability
- **[Crossplane compositions](./crossplane.md)** — infrastructure capabilities this app exposes (none today)
