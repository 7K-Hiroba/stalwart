---
sidebar_position: 2
---

# Container image

Stalwart ships an [official multi-arch container image](https://github.com/stalwartlabs/stalwart/pkgs/container/stalwart) maintained by the upstream project. Following Hiroba's [near-native philosophy](https://hiroba.7kgroup.org/docs/architecture/near-native), this app does **not** maintain its own Dockerfile — the Helm chart consumes the upstream image directly.

| | |
| --- | --- |
| Registry | `docker.io/stalwartlabs/stalwart` (mirror: `ghcr.io/stalwartlabs/stalwart`) |
| Default tag | pinned via `Chart.appVersion` (currently `0.16.1`) |
| Base | `debian:trixie-slim` |
| User | `stalwart` (UID/GID `2000`) |
| Build features | `sqlite postgres mysql rocks s3 redis azure nats enterprise` |

## Why no custom image

The upstream image is well-maintained, signed with Cosign, and built for both `linux/amd64` and `linux/arm64`. Adding a wrapper Dockerfile would mean:

- Tracking upstream releases ourselves
- Re-running their multi-arch build matrix
- Drifting from upstream's security baseline (`setcap cap_net_bind_service`, non-root `stalwart` user)

…with no Hiroba-specific value to add. If you need a custom build (e.g. additional features compiled in), fork upstream's `Dockerfile` rather than overriding through this repo.

## Pinning a different tag

Override `image.tag` in the base chart values:

```yaml
image:
  repository: docker.io/stalwartlabs/stalwart
  tag: "v0.16.1"
```

Leave `tag` empty to track `Chart.appVersion`.

## Runtime expectations

The Helm base chart assumes the upstream image's contract:

| Surface | Value |
| --- | --- |
| Liveness/readiness | `GET /healthz/live` and `/healthz/ready` on port `8080` |
| Config directory | `/etc/stalwart` (mounted from the `config` PVC) |
| Data directory | `/var/lib/stalwart` (mounted from the `data` PVC) |
| Read-only root FS | yes — Stalwart writes only to the mounted volumes |
| Privileged ports | bound by the binary via `cap_net_bind_service`, no Linux capabilities are added to the container |
| Exposed ports | `8080` (HTTP), `25/587/465` (SMTP family), `143/993` (IMAP), `110/995` (POP3), `4190` (ManageSieve) |

If a future Stalwart release changes any of these, update the base chart's probes, port map, or volume mounts to match — don't paper over it with a custom image.
