# HomeLab ArgoCD

GitOps configuration for a two-node Raspberry Pi [k3s](https://k3s.io/) cluster at home, deployed by [Argo CD](https://argo-cd.readthedocs.io/).

This repo is the source of truth for what runs on the cluster. You change the cluster by changing YAML here and merging it to `main`; Argo CD notices the change and applies it. There is no application code, build or test tooling, only Kubernetes manifests, Helm values and Argo CD `Application` definitions.

New here? Read [How it works](#how-it-works) first, then [Making changes](#making-changes).

## The cluster

| Node | Role | Hardware | Storage | Labels |
|---|---|---|---|---|
| `pi5` | control plane (k3s server) | Raspberry Pi 5, 8 GB | SSD mounted at `/data` | `storage=ssd` |
| `raspberrypi4b` | worker (k3s agent) | Raspberry Pi 4, 4 GB, Zigbee adapter, BME280 temperature sensor | SD card only (`/data` is on the SD card) | `hw=zigbee-bridge` |

Node labels are used for scheduling: anything with large or important data is pinned to `pi5` (`storage=ssd`); Zigbee2MQTT must run on the Pi 4 because the Zigbee adapter is plugged in there (`hw=zigbee-bridge`). The BME280 sensor has no label of its own: a node can only have one value for the `hw` key, and it is used for the Zigbee adapter.

Labels are set by hand when a node joins:

```bash
kubectl label node pi5 storage=ssd
kubectl label node raspberrypi4b hw=zigbee-bridge
```

## How it works

```
app-of-apps  (Application, created once by hand, syncs the apps/ folder)
   │
   ├── apps/immich.yml        ──► Helm chart + values from manifests/immich/
   ├── apps/cloudflared.yml   ──► raw manifests from manifests/cloudflared/
   └── ...
```

- **`app-of-apps`** is the root Argo CD `Application`. It points at the `apps/` folder of this repo and creates one child `Application` per file. It is not stored in this repo (see [Bootstrapping](#bootstrapping-a-new-cluster)) and it has **no automated sync**: after adding or removing a file in `apps/`, sync it by hand in Argo CD.
- **`apps/`** holds one Argo CD `Application` per deployed app, all in the `argocd` namespace. Each one points at either:
  - **a Helm chart** from an upstream chart repo, with values taken from this repo through a second `sources` entry (`ref: values`, `$values/manifests/<app>/...`). Examples: [apps/nextcloud.yml](apps/nextcloud.yml), [apps/immich.yml](apps/immich.yml), [apps/Zigbee2Mqtt.yml](apps/Zigbee2Mqtt.yml).
  - **raw manifests** in `manifests/<app>/` through a single `source`. Examples: [apps/cloudflared.yml](apps/cloudflared.yml), [apps/smart-home.yml](apps/smart-home.yml), [apps/mcp-k8s.yml](apps/mcp-k8s.yml).
- **`manifests/`** holds Helm values files and raw manifests, one subdirectory per app, referenced by the matching file in `apps/`.

Every child `Application` follows the same pattern; keep new apps consistent with it:

- `destination.server: https://kubernetes.default.svc` (single in-cluster target),
- `syncPolicy.automated` with `prune: true` and `selfHeal: true`, so Git always wins over manual changes in the cluster,
- `syncOptions: [CreateNamespace=true]`,
- `repoURL: https://github.com/t-tibor/HomeLabArgoCD(.git)`, `targetRevision: main`.

### Sync order

The `argocd.argoproj.io/sync-wave` annotation controls the order in which `app-of-apps` creates apps (lower first, default `0`):

| Wave | Apps | Why |
|---|---|---|
| `-1` | `local-path-provisioner` | storage must exist before anything that needs a volume |
| `0` (unannotated) | `cloudflared`, `smart-home`, `mcp-k8s`, `mcp-prometheus` | |
| `1` | `zigbee2mqtt`, `immich`, `nextcloud` | need storage |

Give a new app a wave if it depends on storage or on another app being ready.

## Apps

| App | Namespace | Source | What it is |
|---|---|---|---|
| `local-path-provisioner` | `local-path-storage` | Kustomize over the upstream manifest (pinned tag) | Dynamic volumes for the default `local-path` StorageClass |
| `cloudflared` | `cloudflared` | raw manifests | Cloudflare Tunnel for reaching services from outside |
| `smart-home` | `smart-home` | raw manifests | Home Assistant, Mosquitto (MQTT), MQTTX web client, light dashboard |
| `zigbee2mqtt` | `smart-home` | Helm chart `zigbee2mqtt` | Zigbee ↔ MQTT bridge, pinned to the Pi 4 |
| `immich` | `immich` | Helm chart `immich` + raw manifests | Photo library (machine learning disabled) with its own Postgres |
| `nextcloud` | `nextcloud` | Helm chart `nextcloud` + raw manifests | File sync and sharing |
| `mcp-k8s` | `mcp-k8s` | raw manifests | Read-only [Kubernetes MCP server](https://github.com/containers/kubernetes-mcp-server) for AI agents |
| `mcp-prometheus` | `mcp-prometheus` | Helm chart `prometheus-mcp-server` + raw manifests | MCP server for querying Prometheus |

Chart versions are pinned in each `apps/*.yml` (`targetRevision`).

### Running on the cluster but not managed here

These were installed by hand and are not reconciled by Argo CD:

- **k3s built-ins**: Traefik (ingress), CoreDNS, metrics-server.
- **Argo CD** itself, in `argocd` (see [Bootstrapping](#bootstrapping-a-new-cluster)).
- **Monitoring**: `kube-prometheus-stack` (Prometheus, Alertmanager, Grafana) in `monitoring`.
- **Headlamp** (cluster UI) in `kube-system`; its admin token secret is [manifests/headlamp/](manifests/headlamp/), applied manually.
- **kubernetes-agent** in `k3s-agent`, an AI agent that uses `mcp-k8s`.
- **Static PersistentVolumes** (see [Storage](#storage)).

## Storage

There are two kinds of volumes:

1. **Dynamic volumes (`local-path`, the default StorageClass).** Created on demand by `local-path-provisioner` under `/data/local-path-storage-provisioner/` on whichever node the pod lands on. On `pi5` that is the SSD; on the Pi 4 it is the SD card. Good for small app data.
2. **Static `hostPath` volumes for important data.** Pre-created PersistentVolumes under `/data/volumes/<app>/` on `pi5`, pinned with node affinity `storage=ssd`, reclaim policy `Retain`. The PVCs in this repo bind to them with `storageClassName: ""` and `volumeName`. Used by Home Assistant, Immich (library and Postgres) and Nextcloud. The PV objects themselves are **not** in this repo; create them by hand before deploying those apps on a new cluster.

k3s ships its own copy of local-path-provisioner. It is **disabled** on this cluster (`disable: [local-storage]` in `/etc/rancher/k3s/config.yaml` on `pi5`), because it fought with the Argo CD-managed one over the shared StorageClass and RBAC objects. Keep it disabled.

## Secrets

Real secret values never go into Git. They live in the gitignored `.local/` folder and are created in the cluster with `kubectl`. Where an app needs a secret, a template with placeholder values sits next to its manifests (e.g. `manifests/openclaw/secret_template.yaml`). For example, `cloudflared` expects a `cloudflared-tunnel-token` secret in its namespace.

## Making changes

1. Create a branch, change the YAML, open a pull request. Branch naming, PR title format and merge rules are in [CLAUDE.md](CLAUDE.md) and apply to humans too. In short: PRs are squash-merged and the PR title, in [Conventional Commits](https://www.conventionalcommits.org/) style, becomes the commit on `main`.
2. After merging, Argo CD applies changes to existing apps within about three minutes. Some changes need a manual step:
   - **Added or removed a file in `apps/`**: sync `app-of-apps` in Argo CD (with *Prune* enabled when removing).
   - **Removed an app**: the child Applications have no cascade-delete finalizer, so deleting one leaves its namespace and resources behind. Delete the namespace by hand. Data on `Retain` volumes stays on disk.
   - **New secrets or static volumes**: create them before the app syncs.
3. Check the result in the Argo CD UI or with `kubectl get applications -n argocd`.

## Bootstrapping a new cluster

1. Install k3s on `pi5` with `/etc/rancher/k3s/config.yaml` containing `disable: [local-storage]`, mount the SSD at `/data`, then join the Pi 4 as an agent. Apply the node labels from [The cluster](#the-cluster).
2. Install Argo CD in the `argocd` namespace and set it up as described in [Argo CD setup](#argo-cd-setup).
3. Create the secrets and static PersistentVolumes the apps need (see [Secrets](#secrets) and [Storage](#storage)).
4. Create the root Application and sync it:

   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: app-of-apps
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/t-tibor/HomeLabArgoCD.git
       targetRevision: HEAD
       path: apps
     destination:
       server: https://kubernetes.default.svc
       namespace: argocd
   ```

## Things to know

- **A folder in `manifests/` doesn't mean the app is deployed.** Only files in `apps/` are deployed. `manifests/openclaw/` is a leftover from an app that was removed.
- **Namespaces** are usually created by `CreateNamespace=true`, but some apps also ship an explicit `Namespace` manifest (e.g. `manifests/smart-home/ns.yaml`). Both work.
- **Cluster-facing tooling is kept locked down.** `mcp-k8s` runs `--read-only --disable-destructive` with a restricted `securityContext`, is bound only to the built-in `view` ClusterRole and has a NetworkPolicy. Keep that posture for similar tools.

## Using the MCP servers

`mcp-k8s` and `mcp-prometheus` give AI agents read-only access to the cluster and its metrics over MCP's streamable HTTP transport.

| Server | In-cluster endpoint | Talks to |
|---|---|---|
| `mcp-k8s` | `http://kubernetes-mcp-server.mcp-k8s.svc.cluster.local:8080/mcp` | Kubernetes API, as the built-in `view` ClusterRole |
| `mcp-prometheus` | `http://mcp-prometheus-prometheus-mcp-server.mcp-prometheus.svc.cluster.local:8080/mcp` | `prometheus-stack-kube-prom-prometheus.monitoring:9090` |

- The Prometheus MCP server relies on the chart's safe defaults: the destructive TSDB admin tools (`snapshot`, `delete_series`, `clean_tombstones`) stay disabled, and it doesn't mount a Kubernetes service account token. Its NetworkPolicy only lets it reach Prometheus and DNS.
- From outside the cluster, port-forward and point the MCP client at `http://localhost:8080/mcp`:

  ```bash
  kubectl port-forward -n mcp-prometheus svc/mcp-prometheus-prometheus-mcp-server 8080:8080
  ```

## Argo CD setup

One-off steps after installing Argo CD. They are not managed by GitOps.

**Run the Argo CD server in insecure mode** (TLS is terminated in front of it):

1. `kubectl edit deployment argocd-server -n argocd`
2. Add `--insecure` to the container args:
   ```yaml
   args:
     - <server executable>
     - --insecure
   ```
3. `kubectl rollout restart deployment argocd-server -n argocd`

**Optional: self-signed certificate for Traefik**

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout homelab.key -out homelab.crt \
  -subj "/CN=homelab.local"

kubectl create secret tls argocd-tls \
  --cert=homelab.crt \
  --key=homelab.key \
  -n argocd
```

## Further docs

Longer how-tos and background notes that don't fit in this README go into the [docs/](docs/) folder, one Markdown file per topic, linked from here:

- [docs/k3s-upgrade.md](docs/k3s-upgrade.md): upgrading k3s on both nodes, one minor version at a time.
