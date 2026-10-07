# helm-rustdesk-server-oss

![Docker Pulls](https://img.shields.io/docker/pulls/rustdesk/rustdesk-server)
![License](https://img.shields.io/badge/license-GPL--3.0-blue)

A [Helm](https://helm.sh/) chart to deploy the [RustDesk](https://rustdesk.com/) self-hosted broker/relay server (OSS) on Kubernetes.

![rustdesk](https://raw.githubusercontent.com/rustdesk/rustdesk/master/flutter/assets/icon.svg)

## Description

RustDesk is a free, open-source, self-hosted remote desktop application (an alternative to TeamViewer or AnyDesk). This chart deploys the RustDesk server components on Kubernetes:

* `hbbs` — the hostname and port discovery **broker** (rendezvous)
* `hbbr` — the **relay** server used for NAT-traversed (hole-punching failed) connections

Both components run in the **same Pod** (as two containers) because they must share the same data directory: the `id_ed25519` key pair generated at first start is used by `hbbs`, and the relay must be able to read it as well.

[https://rustdesk.com](https://rustdesk.com) · [rustdesk/rustdesk-server](https://hub.docker.com/r/rustdesk/rustdesk-server)

## Components & Ports

| Container | Port  | Protocol | Role |
|-----------|-------|----------|------|
| hbbs      | 21115 | TCP      | HTTP API / key exchange |
| hbbs      | 21116 | TCP+UDP  | Port discovery (ID / rendezvous) |
| hbbs      | 21118 | TCP      | KCP / alternative TCP |
| hbbr      | 21117 | TCP      | Relay |
| hbbr      | 21119 | TCP      | Relay (alternative) |

## Prerequisites

* Kubernetes `1.19+`
* A storage option for the shared data volume (default: a `pvc` using the `longhorn` StorageClass; `nfs`, `emptyDir` and `hostPath` are also supported — see [Persistence](#persistence))
* To expose the ports through a public host, [Traefik](https://doc.traefik.io/traefik) with the `IngressRouteTCP`/`IngressRouteUDP` CRDs and matching TCP/UDP entrypoints (see the `ingress` values)

## Installation

Install the chart from the [chart folder](https://github.com/digrouz/helm-rustdesk-server-oss/tree/main/chart):

    helm install rustdesk-server ./chart \
      --set relay.server=<public-relay-host>:21117 \
      --set ingress.hbbs5tcp.enabled=true \
      --set ingress.hbbs6tcp.enabled=true \
      --set ingress.hbbs6udp.enabled=true \
      --set ingress.hbbs8tcp.enabled=true \
      --set ingress.hbbr7tcp.enabled=true \
      --set ingress.hbbr9tcp.enabled=true

A complete [values.yaml](https://github.com/digrouz/helm-rustdesk-server-oss/tree/main/chart/values.yaml) is provided with all options documented inline.

## Configuration

The following table lists the most useful values. Set them with `--set` or in a values file.

| Value | Description | Default |
|-------|-------------|---------|
| `image.repository` | Docker image repository | `rustdesk/rustdesk-server` |
| `image.tag` | Docker image tag | `latest` |
| `relay.server` | Public relay host passed to `hbbs -r` (e.g. `relay.example.com:21117`). Clients use it to reach the relay | `""` (unset) |
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `persistence.data.type` | Data volume type: `pvc`, `nfs`, `emptyDir` or `hostPath` | `pvc` |
| `ingress.*.enabled` | Enable/disable each Traefik `IngressRouteTCP`/`IngressRouteUDP` | `false` |

### `relay.server`

When set, the chart starts `hbbs` with the `-r` flag pointing at this address, e.g. `hbbs -r relay.example.com:21117`. This is the externally-reachable host that clients dial to reach the relay (`hbbr`). If your relay is already exposed on another host you can leave this empty and configure it in the client's server settings instead.

### Persistence

`hbbs` and `hbbr` **must** share the same data directory because the `id_ed25519` key pair generated at first start must be readable by both. The chart therefore mounts a single shared volume at `/root` for both containers inside one Pod.

Because a `ReadWriteOnce` volume cannot be shared between two separate Deployments, the chart runs both components in a single Pod and fixes `replicaCount` to `1`. Supported `persistence.data.type` values:

* `pvc` (default) — a `PersistentVolumeClaim` is created (or an existing one can be supplied via `persistence.data.existingClaim` / `persistence.data.existing`)
* `nfs` — a shared NFS volume, useful when the data must live on a network share
* `emptyDir` — ephemeral (the keys are regenerated on every restart)
* `hostPath` — a directory on the node's filesystem

## Notes

* `replicaCount` is intentionally fixed to `1`: the shared `id_ed25519` key pair and relay state must not be duplicated across replicas.
* The chart emits Traefik `IngressRouteTCP`/`IngressRouteUDP` resources using the `traefik.io/v1alpha1` API group. The `terminationDelay` field is set at the service level, matching the `ServiceTCP` schema.
* Inbound ports that are not covered by an ingress route (when all `ingress.*.enabled` are `false`) can be exposed through the Kubernetes Service (`service.type: NodePort` / `LoadBalancer`) instead.
* The `rustdesk-server` image runs the server as root inside the container; the container does not ship an HTTP health endpoint, so TCP socket probes are used.

## Issues

If you encounter an issue please open a ticket at [github](https://github.com/digrouz/helm-rustdesk-server-oss/issues)
