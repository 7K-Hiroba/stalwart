# stalwart

Stalwart

## Structure

```
├── helm/
│   ├── base/           # Core k8s resources (Deployment, Service, Ingress)
│   └── platform/       # Platform dependencies with provider switching
├── compositions/
│   └── crossplane/     # App-specific Crossplane compositions (XRDs, Compositions)
├── gitops/
│   ├── argocd/         # ArgoCD Application manifests
│   └── fluxcd/         # FluxCD Kustomization manifests
├── docs/               # TechDocs content
├── .github/workflows/  # CI/CD (references 7K-Hiroba/workflows-library)
├── Dockerfile
└── catalog-info.yaml   # Backstage catalog entry
```

## Quick Start

```bash
# Build the container image
docker build -t stalwart:dev .

# Deploy base application
helm install stalwart ./helm/base

# Deploy platform dependencies (optional)
helm install stalwart-platform ./helm/platform
```

## Documentation

Full documentation is available at [hiroba.7kgroup.org/apps/stalwart](https://hiroba.7kgroup.org/apps/stalwart), or locally under `docs/`.

## Part of the Hiroba ecosystem

Scaffolded with [Hiroba](https://github.com/7K-Hiroba/Hiroba) by 7KGroup.
