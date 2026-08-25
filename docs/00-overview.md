# Architecture overview

A Kairos node treated as a declarative artifact: the OS image and the runtime configuration both live in this GitHub repo, and the Pi converges to them.

## Components

```
   GitHub repo (shashankpai/kairos-test)
        |
        |--- image/Dockerfile ----------------------+
        |                                            |
        |--- .github/workflows/build-and-publish.yaml|  GitHub Actions
        |                                            |  builds linux/arm64
        v                                            v
   GHCR: ghcr.io/shashankpai/kairos-pi:<tag>  <-- pushed image
        |
        | (manual SSH: sudo kairos-agent upgrade --source oci:...)
        v
   Raspberry Pi 4 (arm64, USB pen drive ~64 GB)
        |
        |  on every boot, stages.boot.git pulls:
        |    https://github.com/shashankpai/kairos-test.git -> /oem/cloud-config-files
        v
   /oem/*.yaml applied in lexicographic order by Kairos cloud-init
```

## Data flow

1. **Image build.** A push to `main` (or a `v*` tag) triggers GitHub Actions, which builds `image/Dockerfile` for `linux/arm64` and pushes to GHCR. Tagged builds also produce a `*.raw` artifact for fresh-node flashing.
2. **OS upgrade.** You SSH into the Pi and run `sudo kairos-agent upgrade --source oci:ghcr.io/shashankpai/kairos-pi:<tag>`, then reboot. Kairos writes the new image to the passive slot atomically and swaps on reboot. If the new image fails to boot, Kairos automatically falls back to the previous (now-passive) image.
3. **Config change.** You edit a file under `cloud-config/` and push to `main`. On the Pi's next boot, the `stages.boot.git` entry (defined in `cloud-config/10_git-pull.yaml` and baked into the flash-time config) clones this repo into `/oem/cloud-config-files`. Kairos then applies every `/oem/*.yaml` in lexicographic order.

## Why this shape

- **Immutable OS.** The rootfs is read-only; the only persistent state is `/oem` (cloud-config) and `/usr/local` (persistent data). Upgrades replace the whole OS image atomically — no in-place package drift.
- **Config-as-code.** All runtime config lives in git, not on the box. The Pi pulls it on boot, so a reboot is the convergence mechanism.
- **Manual upgrades.** For a single edge node, SSH-triggered upgrades are simpler and more auditable than a scheduled timer or a Kubernetes operator. The workflow can be promoted to GitHub-Actions-driven SSH or to the Kairos operator later without changing the image or config.
- **GHCR.** Free, tied to the GitHub identity, no extra account. The `GITHUB_TOKEN` secret is auto-provided to Actions, so no registry credentials to manage.

## Boundaries

- This repo does **not** manage k3s workloads — only the OS and node-level config. k3s manifests belong elsewhere (or in a future `k3s/` directory here).
- This repo does **not** manage multiple nodes. The cloud-config and runbook assume one Pi. Multi-node would require k3s server/agent config and a shared token.
