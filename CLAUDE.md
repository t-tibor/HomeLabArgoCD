# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

GitOps configuration for a home Kubernetes cluster (Raspberry Pi 5 master + Raspberry Pi 4 agent, see [Nodes.md](Nodes.md)), managed by ArgoCD. There is no build/test/lint tooling — this repo is pure Kubernetes YAML (raw manifests + Helm values), applied to the cluster by ArgoCD watching this git repo. "Development" means editing YAML and letting ArgoCD reconcile it, or applying files manually with `kubectl`/`helm` for pieces not yet wired into ArgoCD (see below).

## Repo structure

- **`apps/`** — flat list of ArgoCD `Application` CRs, one per deployed app. Each `Application` lives in the `argocd` namespace and points at either:
  - a Helm chart from a remote repo, with values sourced from this repo via a second `sources` entry (`ref: values`, `$values/manifests/<app>/...`), e.g. [apps/nextcloud.yml](apps/nextcloud.yml), [apps/immich.yml](apps/immich.yml), [apps/Zigbee2Mqtt.yml](apps/Zigbee2Mqtt.yml); or
  - raw manifests directly under `manifests/<app>/` via a single `source`, e.g. [apps/cloudflared.yml](apps/cloudflared.yml), [apps/smart-home.yml](apps/smart-home.yml), [apps/n8n.yml](apps/n8n.yml).
- **`manifests/`** — Helm values files and raw Kubernetes manifests, one subdirectory per app, referenced by the matching file in `apps/`.
- **`.local/`** — gitignored, holds real secret values (e.g. admin passwords) kept out of source control; templated/placeholder secrets live under `manifests/` instead (e.g. `manifests/openclaw/secret_template.yaml`).
- **`ManualSetup.md`** — one-off manual cluster bootstrap steps (ArgoCD insecure mode, self-signed TLS cert for Traefik) that aren't expressed as GitOps manifests.
- **`Nodes.md`** — physical node inventory and hardware labels used for node selectors/affinity.

Every `Application`:
- sets `destination.server: https://kubernetes.default.svc` (single in-cluster target — there's no multi-cluster setup here),
- sets `syncPolicy.automated.{prune,selfHeal}: true` and `syncOptions: [CreateNamespace=true]`,
- targets `repoURL: https://github.com/t-tibor/HomeLabArgoCD(.git)`, `targetRevision: main`.

Keep new apps consistent with this pattern unless there's a specific reason to deviate.

## Sync ordering

Deployment order across apps is controlled with the `argocd.argoproj.io/sync-wave` annotation (lower syncs first; apps without the annotation default to wave `0`):

- `local-path-provisioner` → wave `-1` (storage provisioner must exist before anything else)
- `zigbee2mqtt`, `immich`, `n8n`, `nextcloud` → wave `1`
- `cloudflared`, `smart-home` → unannotated (wave `0`)

When adding an app that depends on storage or another app being ready, set its sync-wave accordingly.

## Important quirks to know before editing

- **Orphaned manifests can outlive their app.** `manifests/openclaw/` still exists on disk, but `apps/openclaw.yml` was deleted (see git history: "Remove openclaw app") — the app is no longer deployed even though its manifests remain. Don't assume presence under `manifests/` implies an active deployment; cross-check `apps/`.
- Namespaces are generally created via `CreateNamespace=true` in the Application's syncOptions, but some apps (e.g. `smart-home`) also ship an explicit `Namespace` manifest (`manifests/smart-home/ns.yaml`) — both approaches exist in this repo.
- Security-conscious defaults are used for cluster-facing services (e.g. `manifests/mcp/all.yaml` runs `kubernetes-mcp-server` as `--read-only --disable-destructive` with a locked-down `securityContext`, bound to the built-in `view` ClusterRole) — preserve this posture when adding similar tooling.
