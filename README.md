# Lab GitOps

GitOps-managed Kubernetes lab on Proxmox using [Flux](https://fluxcd.io/).

## Architecture

- **Cluster:** k3s v1.36 (single node) on Proxmox LXC
- **PostgreSQL 18** — CloudNativePG operator (managed via HelmRelease)
- **Redis 8** — StatefulSet with persistent storage
- **MinIO** — S3-compatible object storage (NodePort 31900 / console 31901)
- **Uptime Kuma** — uptime monitoring
- **Homepage** — dashboard front door
- **Monitoring** — kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
- **Ingress:** Traefik (built into k3s)
- **Notifications:** Flux → Discord webhook

## Repo layout

```
├── apps/                 # Workloads
│   ├── homepage/         #   Dashboard (home.azzam-lab)
│   ├── minio/            #   S3 storage (minio.azzam-lab)
│   ├── postgres/         #   PG cluster (CNPG)
│   ├── redis/            #   Redis StatefulSet + Service
│   └── uptime-kuma/      #   Uptime monitoring (kuma.azzam-lab)
├── infra/                # Infrastructure / operators
│   ├── cnpg/             #   CloudNativePG operator (Helm)
│   ├── monitoring/       #   kube-prometheus-stack (Prometheus+Grafana)
│   ├── notifications/    #   Discord Provider + Alert
│   └── secrets/          #   SOPS-encrypted secrets
├── clusters/production/  # Flux Kustomizations (entry points)
└── kustomization.yaml
```

## How it works

Flux watches this repo and reconciles the cluster to match it.
Changes are made by **pushing to `main`** — never by editing the cluster directly.

- `apps` Kustomization → `./apps`
- `infra` Kustomization → `./infra`
- Reconcile interval: 10m (or `flux reconcile kustomization <name>` for instant)

## Pods

Current workloads running in the cluster (single node `docker`, k3s v1.36):

| Namespace | Pod | Purpose |
|---|---|---|
| `app-dev` | `homepage-*` | Dashboard front door |
| `app-dev` | `uptime-kuma-*` | Uptime monitoring |
| `app-dev` | `hello-test-*` | Test app (can be removed) |
| `lab-infra` | `pg-lab-1` | PostgreSQL 18 primary (CNPG) |
| `lab-infra` | `redis-0` | Redis 8 + metrics exporter |
| `storage` | `minio-*` | S3-compatible object storage |
| `monitoring` | `prometheus-*` | Prometheus (metrics collection) |
| `monitoring` | `kube-prometheus-stack-grafana-*` | Grafana dashboards |
| `monitoring` | `alertmanager-*` | Alert routing |
| `monitoring` | `kube-prometheus-stack-*` | Operator, kube-state-metrics, node-exporter |
| `cnpg-system` | `cnpg-cloudnative-pg-*` | CloudNativePG operator |
| `flux-system` | `helm-controller` / `kustomize-controller` / `source-controller` / `notification-controller` | Flux controllers |
| `flux-system` | `flux-discord-bridge-*` | Optional commit-message bridge (dormant) |
| `kube-system` | `traefik-*`, `coredns-*`, `local-path-provisioner-*`, `metrics-server-*`, `svclb-*` | k3s built-ins |

## Access (LAN)

| Service | URL |
|---|---|
| Homepage | `http://home.azzam-lab` |
| Grafana | `http://grafana.azzam-lab` |
| Uptime Kuma | `http://kuma.azzam-lab` |
| MinIO console | `http://minio.azzam-lab` |
| MinIO S3 API | `http://192.168.0.28:31900` |
| PostgreSQL | `pg-lab-rw.lab-infra.svc:5432` |
| Redis | `redis.lab-infra.svc:6379` |

*(Hostnames resolve via Pi-hole local DNS — add `192.168.0.35` as your DNS server.)*

## Secrets & SOPS

Secrets are encrypted with [SOPS](https://getsops.io/) + [Age](https://age-encryption.org/) and stored in `infra/secrets/*.sops.yaml`. Flux decrypts them on deploy using the Age private key (stored as the `sops-age` Kubernetes Secret).

**Encrypted secrets:**
- `minio-credentials` (MinIO root user/password)
- `grafana-admin` (Grafana admin credentials)
- `discord-url` (Flux Discord webhook)

**To edit a secret:**
```bash
sops edit infra/secrets/minio.sops.yaml
```
*(Requires the Age private key — `~/.config/sops/age/keys.txt` on the cluster host.)*

**Never commit plaintext secrets.** If a secret leaks (e.g. appears in git history), **rotate it** — see the git history for examples of why this matters.

## Backups

- **PostgreSQL:** nightly `pg_dump` at 02:00 (keeps 14)
- **LXC snapshot:** nightly Proxmox snapshot at 02:30 (keeps 7)

## Notes

- k3s runs in an unprivileged LXC — requires `--kubelet-arg=feature-gates=KubeletInUserNamespace=true`
- Secrets are managed by Flux (SOPS); never `kubectl edit` them directly
