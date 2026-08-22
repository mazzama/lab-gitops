# Lab GitOps

GitOps-managed Kubernetes lab on Proxmox using [Flux](https://fluxcd.io/).

## Architecture

- **Cluster:** k3s v1.36 (single node) on Proxmox LXC
- **PostgreSQL 18** — CloudNativePG operator (managed via HelmRelease)
- **Redis 8** — StatefulSet with persistent storage
- **Ingress:** Traefik (built into k3s)
- **Notifications:** Flux → Discord webhook

## Repo layout

```
├── apps/                 # Workloads
│   ├── postgres/         #   PG cluster (CNPG)
│   └── redis/            #   Redis StatefulSet + Service
├── infra/                # Infrastructure / operators
│   ├── cnpg/             #   CloudNativePG operator (Helm)
│   └── notifications/    #   Discord Provider + Alert (+ optional bridge)
├── clusters/production/  # Flux Kustomizations (entry points)
└── kustomization.yaml
```

## How it works

Flux watches this repo and reconciles the cluster to match it.
Changes are made by **pushing to `main`** — never by editing the cluster directly.

- `apps` Kustomization → `./apps`
- `infra` Kustomization → `./infra`
- Reconcile interval: 10m (or `flux reconcile kustomization <name>` for instant)

## Access

| Service | Endpoint |
|---|---|
| PostgreSQL (rw) | `pg-lab-rw.lab-infra.svc:5432` |
| Redis | `redis.lab-infra.svc:6379` |
| Ingress | `http://<node-ip>/` |

Secrets (webhook URLs, DB passwords) are stored as Kubernetes Secrets, never in this repo.
