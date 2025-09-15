# HomeLab ArgoCD

This repo contains the GitOps configuration for my Raspberry Pi Kubernetes cluster managed with [ArgoCD](https://argo-cd.readthedocs.io/).

## Structure

```
.
├── apps/                        # Flat list of ArgoCD Applications (App CRs)
│   ├── app-of-apps.yaml          # Root "App of Apps"
│   ├── local-path-provisioner.yaml
│   └── nextcloud.yaml
└── manifests/                    # Helm values or raw manifests for each app
    ├── local-path-provisioner/
    │   └── values.yaml
    └── nextcloud/
        └── values.yaml
```

- **`apps/`** → Contains only ArgoCD `Application` resources.
- **`manifests/`** → Contains Helm values and Kubernetes manifests for each app.

## Sync Order

ArgoCD **sync waves** ensure apps are deployed in the right order:

- `local-path-provisioner` → **wave 0**
- `nextcloud` → **wave 1**

This guarantees the storage provisioner is available before Nextcloud starts.

## Installed Apps

- **local-path-provisioner**  
  Provides dynamic PersistentVolume provisioning using the SSD storage on the Raspberry Pi.

- **nextcloud**  
  Self-hosted file sync and sharing platform. Uses the `local-path` storage class.

## Usage

1. Bootstrap ArgoCD and point it at this repo.
2. Deploy the App-of-Apps (`apps/app-of-apps.yaml`).
3. ArgoCD will:
   - Install the storage provisioner first.
   - Deploy Nextcloud afterwards.

## Notes

- The local-path-provisioner writes data to `/opt/local-path-provisioner` on each node.  
  Make sure this directory exists on the Raspberry Pi SSD.
- Default StorageClass is set to `local-path`.
