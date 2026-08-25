# kairos-test

Immutable Kubernetes edge node running [Kairos](https://kairos.io) on a Raspberry Pi 4 (arm64) with k3s, managed entirely from this GitHub repository.

## Goal

A single source of truth for an edge Kairos node:

- **OS image** — a custom Kairos image extending upstream `hadron`, built in GitHub Actions and published to GHCR.
- **Configuration** — cloud-config files in this repo are pulled onto the Pi on every boot via Kairos's `stages.boot.git`, so config changes land with a reboot, no re-flashing.
- **Upgrades** — atomic A/B OS upgrades triggered manually over SSH against the GHCR image, with automatic fallback to the passive image on boot failure.

## Repo layout

```
kairos-test/
  cloud-config/             # canonical cloud-config files (pulled by the Pi at boot)
    00_base.yaml            # users, ssh keys, k3s
    10_git-pull.yaml        # boot-time git pull of this repo into /oem
  image/
    Dockerfile              # custom Kairos image extending upstream hadron
  .github/workflows/
    build-and-publish.yaml  # build arm64 image -> GHCR; on tags, also emit a .raw
  docs/
    00-overview.md          # architecture
    01-flashing.md          # how the SD/SSD was initially flashed
    02-upgrades.md          # manual SSH upgrade runbook + rollback
    03-config-management.md # how config changes reach the Pi
    04-troubleshooting.md   # debugging and recovery
  build/                    # local-only flash artifacts (gitignored: *.raw, *.img, cloud-config.yaml)
```

## Current state

| Item | Value |
| --- | --- |
| Upstream base | `quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3sv1.36.3+k3s1` |
| Custom image | `ghcr.io/shashankpai/kairos-pi:<tag>` (see [docs/02-upgrades.md](docs/02-upgrades.md)) |
| Target | Raspberry Pi 4, arm64, USB pen drive (~64 GB) |
| Kubernetes | k3s (single-node edge) |

## Documentation

- [Architecture overview](docs/00-overview.md)
- [Flashing a node](docs/01-flashing.md)
- [Upgrades & rollback](docs/02-upgrades.md)
- [Config management](docs/03-config-management.md)
- [Troubleshooting](docs/04-troubleshooting.md)

## Secrets hygiene

**Never commit secrets.** The flash-time `build/cloud-config.yaml` may contain the initial user password and (if the repo stays private) a GitHub deploy key. That file is gitignored. Runtime config in `cloud-config/` must be secret-free — use SSH keys for access, not passwords. See [docs/01-flashing.md](docs/01-flashing.md).
