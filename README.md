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
    01_ssd_storage.yaml     # SSD reconciliation + local-path provisioner config
    10_git-pull.yaml        # boot-time curl+tar pull of this repo into /oem
    20_k8s_workloads.yaml   # Kairos stages to apply k8s manifests
  k8s/                      # k8s manifests (pulled by git-pull, applied by workloads stage)
    namespace.yaml          # monitoring namespace
    prometheus.yaml         # Prometheus + ConfigMap + PVC
    node-exporter.yaml      # node-exporter DaemonSet
    kube-state-metrics.yaml # kube-state-metrics Deployment
    grafana.yaml            # Grafana + ConfigMaps + PVC
  image/
    Dockerfile              # custom Kairos image extending upstream hadron
  .github/workflows/
    build-and-publish.yaml  # build arm64 image -> GHCR; on tags, also emit a .raw
  docs/
    00-overview.md          # architecture
    phase-1-flashing.md     # how to flash the Pi (includes SSD setup)
    phase-2-github-managed.md # GitHub Actions builds, boot-time config pull, upgrades
    phase-3-monitoring-fleet.md # monitoring stack, fleet, automated upgrades
    phase-3-ssd-storage.md  # SSD configuration (hybrid approach)
    phase-4-opnsense-network.md # network migration to VLAN 10
  build/                    # local-only flash artifacts (gitignored: *.raw, *.img, cloud-config.yaml)
```

## Current state

| Item | Value |
| --- | --- |
| Upstream base | `quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1` |
| Custom image | `ghcr.io/shashankpai/kairos-pi:<tag>` (see [docs/phase-2-github-managed.md](docs/phase-2-github-managed.md)) |
| Target | Raspberry Pi 4, arm64, USB pen drive (~64 GB) + USB SSD |
| Kubernetes | k3s (single-node edge, data on SSD) |
| Network | VLAN 10 (10.0.0.34), OPNsense routing |
| Monitoring | Prometheus + Grafana + node-exporter (on SSD) |

## Documentation

- [Architecture overview](docs/00-overview.md)
- [Phase 1: Flashing & Initial Boot](docs/phase-1-flashing.md) — includes SSD setup
- [Phase 2: GitHub-Managed Image & Config](docs/phase-2-github-managed.md)
- [Phase 3: Monitoring Stack & Fleet](docs/phase-3-monitoring-fleet.md)
- [Phase 3: SSD-Backed Storage](docs/phase-3-ssd-storage.md) — hybrid approach
- [Phase 4: Network Migration to VLAN 10](docs/phase-4-opnsense-network.md)

## Secrets hygiene

**Never commit secrets.** The flash-time `build/cloud-config.yaml` may contain the initial user password and (if the repo stays private) a GitHub deploy key. That file is gitignored. Runtime config in `cloud-config/` must be secret-free — use SSH keys for access, not passwords. See [docs/01-flashing.md](docs/01-flashing.md).
