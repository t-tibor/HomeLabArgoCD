# CLAUDE.md

Rules for working in this repo. For what the repo is, how it is laid out, which apps run on the cluster and how they are deployed, read [README.md](README.md).

## Branching

- Never commit to `main` directly. Every change goes through a pull request.
- One branch per logical change, named `<type>/<short-kebab-description>`, e.g. `fix/local-path-default-storageclass`, `chore/remove-n8n`.

## Pull requests and commits

- PRs are **squash-merged**, and the **PR title becomes the commit message on `main`**. The PR title is therefore what matters most.
- PR titles follow [Conventional Commits](https://www.conventionalcommits.org/): `<type>(<optional scope>): <description>`
  - `type`: `feat`, `fix`, `chore`, `docs`, `refactor`, `ci`, `perf`, `revert`
  - `scope`: the app name as used in `apps/` (e.g. `immich`, `local-path-provisioner`); omit it for repo-wide changes
  - `description`: imperative, lower case, no trailing period, e.g. `fix(immich): raise postgres memory limit`
- Use the same style for the commit messages on the branch.
- The PR description says **why** the change is needed, **what** changed and any **manual steps after merging** (see README → *Making changes*).

## Changes

- Follow the conventions in README.md (Application pattern, sync waves, storage) unless there is a specific reason to deviate; say so in the PR if you do.
- Where a change can be checked offline, check it before opening the PR (e.g. `kustomize build` for Kustomize directories and diff the output against `main`).
- Never commit real secrets. They live in the gitignored `.local/`; only templates go under `manifests/`.
- Keep cluster-facing tooling locked down (read-only, minimal RBAC, restricted `securityContext`), as in `manifests/mcp-k8s/`.
- When an app is added, removed or materially changed, update README.md in the same PR.
