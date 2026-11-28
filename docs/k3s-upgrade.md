# Upgrading k3s

How to upgrade k3s on both nodes. Last done on 2026-11-28, from v1.33 to **v1.37.2+k3s1**, in about 20 minutes, without any app restarts.

## How it works

- **One minor version at a time.** Kubernetes doesn't support skipping minor versions, so 1.33 → 1.37 is four steps: 1.34, 1.35, 1.36, 1.37.
- **In each step, pi5 (server) first, then the Pi 4 (agent)**, and both nodes on the same version before the next step.
- **Each step re-runs the k3s install script with the channel for that minor version** (`INSTALL_K3S_CHANNEL=v1.35` etc.), which installs that version's latest patch release. Find the current stable version at <https://docs.k3s.io/release-notes/> or with `curl -s https://update.k3s.io/v1-release/channels | grep -o '"stable"[^}]*'`.
- **Don't drain the nodes.** The apps are pinned (SSD data on pi5, Zigbee adapter on the Pi 4) and have nowhere else to go. They don't need to move anyway: when k3s restarts, running containers keep going.

## Before you start

Run these once.

1. **cgroup v2 on both nodes.** Since Kubernetes 1.35 the kubelet refuses to start on cgroup v1.
   ```bash
   stat -fc %T /sys/fs/cgroup/   # must print cgroup2fs
   ```
2. **Deprecated APIs.**
   ```bash
   kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
   ```
   Lines with an empty `removed_release` are only deprecated and are fine. Anything with a `removed_release` at or below the target version must be fixed first.
   On 2026-11-28 only core `v1 endpoints` and `hub.traefik.io/v1alpha1 apiaccesses` showed up, both without a removal date.
3. **Traefik customisations.**
   ```bash
   kubectl get helmchartconfig -A
   ```
   If this lists anything, check it against the release notes of the Traefik version the target k3s ships (see the k3s release notes table). There were none on 2026-11-28.
4. **Agent join settings.** On the Pi 4, `sudo cat /etc/systemd/system/k3s-agent.service.env` must show `K3S_URL` and `K3S_TOKEN`. The agent command below reads them from there. **Never run the install script on the Pi 4 without `K3S_URL`, or it sets the Pi 4 up as a second server.**
5. **App backups.** At least the Immich Postgres, Home Assistant and Nextcloud databases.
6. **Free disk on the Pi 4**, with `df -h /`. Every step pulls new system images onto its SD card.

## Each step

Replace `v1.XX` with the minor version of this step.

**1. Back up the k3s database on pi5.** The single server uses SQLite (no etcd). Containers keep running while k3s is stopped.
```bash
sudo systemctl stop k3s
sudo tar czf ~/k3s-backup-$(date +%F-%H%M).tgz /var/lib/rancher/k3s/server/db /var/lib/rancher/k3s/server/token /etc/rancher/k3s
sudo systemctl start k3s
```

**2. Upgrade pi5.** It was installed without extra options; its only setting, `disable: [local-storage]`, lives in `/etc/rancher/k3s/config.yaml`, which the install script doesn't touch.
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.XX sh -
```

**3. Check.** Wait until things settle, usually under two minutes:
```bash
kubectl get nodes -o wide                            # pi5 on the new version, both Ready
kubectl get pods -A | grep -vE 'Running|Completed'   # nothing stuck
kubectl get applications -n argocd                   # Synced / Healthy
```

**4. Upgrade the Pi 4.** This reuses its existing join settings:
```bash
sudo sh -c '. /etc/systemd/system/k3s-agent.service.env && curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.XX K3S_URL="$K3S_URL" K3S_TOKEN="$K3S_TOKEN" sh -'
```

**5. Check again** with the same commands. Both nodes should show the same version. Then open Home Assistant, Zigbee2MQTT, Immich and Nextcloud.

## What's normal during an upgrade

All of these showed up on 2026-11-28 and cleared up by themselves:

- **The API is down for 30–60 seconds** while pi5 restarts.
- **k3s replaces its bundled parts** (CoreDNS, metrics-server, Traefik, the `svclb-traefik` pods) in some steps. New pods show up with new hash names and ages of a few seconds.
- **`helm-install-traefik` fails once or twice before completing.** It runs before the API is ready.
- **Short health-check failures for pods that talk to the API**, like `local-path-provisioner` and `kube-state-metrics` (`Readiness`/`Liveness probe failed ... 503`), while pi5 restarts.
- **metrics-server failing readiness** (`statuscode: 500`) for a minute after each kubelet restart.
- **`InvalidDiskCapacity: invalid capacity 0 on image filesystem`**, logged once when the Pi 4's kubelet starts.

A pod that keeps restarting, or a node that stays `NotReady`, is **not** normal. Stop there and look at `journalctl -u k3s` (pi5) or `journalctl -u k3s-agent` (Pi 4).

## Version notes

- **1.34.13:** brought the big Traefik jump, 3.3.6 → 3.7.15, earlier than expected. Traefik logs new warnings recommending optional settings (`SafeNaming`, `aliasHeadersStrategy`, encoded characters). The old behaviour is kept, so nothing had to change.
- **1.35:** the kubelet requires cgroup v2.
- **1.37.2:** installs the Kubernetes Gateway API resource types (optional, unused here) and restarts Traefik with updated settings. The old Traefik pod serves traffic until the new one is ready.

## If a step fails

k3s can't be downgraded in place. On pi5:

1. `sudo systemctl stop k3s`
2. Restore the backup from the start of the failed step:
   ```bash
   sudo rm -rf /var/lib/rancher/k3s/server/db
   sudo tar xzf ~/k3s-backup-<timestamp>.tgz -C /
   ```
3. Reinstall the previous version:
   ```bash
   curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.<previous> sh -
   ```

If the Pi 4 was already upgraded in that step, reinstall the previous version there too, with the agent command above.

## Afterwards

- Keep the newest `~/k3s-backup-*.tgz` on pi5 for a few days, then delete the backups.
- Old system images are cleaned up automatically by the kubelet's image garbage collection.
- Check that Argo CD's version supports the new Kubernetes version, and upgrade it if not. It isn't managed by this repo; see the README.
- Update the "Last done" line at the top of this file.
