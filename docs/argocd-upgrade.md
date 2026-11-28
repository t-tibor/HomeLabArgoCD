# Upgrading Argo CD

How to upgrade Argo CD itself. Argo CD isn't managed by this repo: it was installed by hand from the upstream non-HA `install.yaml`. Last done on 2026-11-28, from v3.1.9 to **v3.6.2**, without any app downtime.

## How it works

- **Minor versions can be skipped**, unlike Kubernetes. You must read the official upgrade notes for **every** minor version in between, under [Upgrading](https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/overview/) → "v3.x to v3.y". Then apply the target version's manifest in one go.
- **Always upgrade with server-side apply:** `--server-side --force-conflicts`. Since 3.3 the ApplicationSet CRD is too large for plain `kubectl apply`, and it fails with `metadata.annotations: Too long`.
- **The upgrade replaces Argo CD's own Deployments, StatefulSet and CRDs** with the upstream versions. Anything changed by hand in those (args, env, resources) is overwritten. Settings belong in the ConfigMaps (`argocd-cm`, `argocd-cmd-params-cm`, `argocd-rbac-cm`), which the upgrade leaves alone.
- **Your apps aren't affected.** Workloads keep running while Argo CD restarts; only syncing from Git pauses for a minute or two. Applications and AppProjects are plain Kubernetes objects and stay as they are.

## Before you start

1. **Find the current version and the target.**
   ```bash
   kubectl -n argocd get deploy argocd-server -o jsonpath='{.spec.template.spec.containers[0].image}'
   ```
   For the latest release, see <https://github.com/argoproj/argo-cd/releases>. Stay on one of the three most recent minor versions: older ones get no more fixes.
2. **Check the Kubernetes version** against the target release's "Tested versions" table in the [installation docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/).
3. **Read every upgrade guide between the current and target version** and note anything that touches this setup: Helm and OCI charts (`mcp-prometheus` uses an OCI chart from `ghcr.io`), Kustomize (`local-path-provisioner`), RBAC, SSO, notifications, tracing.
4. **Check that all apps are Synced and Healthy**, so you can tell afterwards whether the upgrade broke anything:
   ```bash
   kubectl get applications -n argocd
   ```
5. **Check that nothing was changed by hand in the Deployments.** The only intentional setting is `server.insecure`, kept in `argocd-cmd-params-cm` (see the README's *Argo CD setup*):
   ```bash
   kubectl -n argocd get cm argocd-cmd-params-cm -o jsonpath='{.data}'                                # expect {"server.insecure":"true"}
   kubectl -n argocd get deploy argocd-server -o jsonpath='{.spec.template.spec.containers[0].args}'  # expect ["/usr/local/bin/argocd-server"]
   ```
   If the args contain anything else, move it into `argocd-cmd-params-cm` first, or it will be lost.

## Upgrade

**1. Back up the Argo CD objects.** The file contains secrets (repo credentials, admin password), so keep it safe and delete it afterwards.
```bash
kubectl get -n argocd applications,appprojects,configmaps,secrets -o yaml > ~/argocd-backup-$(date +%F).yaml
```

**2. Apply the target version:**
```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.X.Y/manifests/install.yaml
```

**3. Wait for the rollout:**
```bash
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd rollout status deploy/argocd-repo-server
kubectl -n argocd rollout status statefulset/argocd-application-controller
```

## Check

```bash
kubectl -n argocd get pods                                  # all Running, no restarts
kubectl -n argocd get deploy,statefulset -o wide            # all argocd images on the new version
kubectl get applications -n argocd                          # same Synced / Healthy state as before
kubectl -n argocd logs deploy/argocd-server | grep serving  # "... serving on port 8080 (... tls: false ...)"
```

`tls: false` shows the insecure setting is still in effect. Then open the Argo CD UI the usual way.

Warnings in the server log about unconfigured webhook secrets (GHCR, DockerHub) are expected. Webhooks aren't used; Argo CD polls Git.

## If something goes wrong

Re-apply the previous version's manifest with the same flags:
```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v<previous>/manifests/install.yaml
```
If Applications or settings were lost, restore them from the backup with `kubectl apply -f ~/argocd-backup-<date>.yaml`.

## Afterwards

- Delete `~/argocd-backup-*.yaml`.
- Update the "Last done" line at the top of this file, and add notes below if a version needed special handling.

## Version notes

- **3.1.9 → 3.6.2 (2026-11-28):**
  - The upgrade was done in one jump.
  - The only change needed beforehand was moving `--insecure`, which had been added to `argocd-server`'s args with `kubectl edit`, into `argocd-cmd-params-cm` as `server.insecure: "true"`. The upgrade did reset the args.
  - None of the breaking changes in between affected this setup:
    - 3.2: hydration paths.
    - 3.3: server-side apply now required.
    - 3.4: cluster version format.
    - 3.5: Helm 4 with plain-HTTP OCI registries, GnuPG replaced, React 19 UI.
    - 3.6: OpenTelemetry settings.
  - Dex went to v2.46.0 and Redis to 8.10.1 along the way.
