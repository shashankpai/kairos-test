# Upgrades & rollback

How to move the Pi to a new OS image, validate it, and recover if it goes wrong.

## Image source

Custom images are built by GitHub Actions (see [`.github/workflows/build-and-publish.yaml`](../.github/workflows/build-and-publish.yaml)) and published to:

```
ghcr.io/shashankpai/kairos-pi:<tag>
```

Available tags:
- `:latest` — last build from `main`.
- `:sha-<short>` — pinned to a specific commit on `main`.
- `:vX.Y.Z` — a release tag (also produces a `.raw.gz` flash artifact on the GitHub Release).

List what's on the Pi now and what's available:

```bash
ssh kairos@<pi>
sudo kairos-agent upgrade list-releases
```

## Upgrade the active system

```bash
ssh kairos@<pi>
sudo kairos-agent upgrade --source oci:ghcr.io/shashankpai/kairos-pi:v0.1.0
sudo reboot
```

Kairos downloads the image, writes it to the **passive** slot, and makes it active on next boot. The currently-running image becomes the new passive (fallback). The upgrade is atomic — it either fully replaces the OS or doesn't touch it.

## Post-reboot validation

```bash
ssh kairos@<pi>
# Confirm the new image is active
sudo kairos-agent upgrade list-releases | head -n5
# k3s came back up
sudo k3s kubectl get nodes    # k3s embeds kubectl; kubeconfig is root-only
# Any package added in image/Dockerfile is present
# e.g.: figlet --version
# Custom version stamp from the Dockerfile
grep KAIROS_TEST_VERSION /etc/os-release
```

Only proceed to the recovery upgrade once the active system is confirmed healthy.

## Upgrade the recovery system

The recovery image is a separate, minimal boot environment used to repair the active system. Keep it close to the active version. **Only upgrade recovery after the active upgrade is verified.**

```bash
ssh kairos@<pi>
sudo kairos-agent upgrade --recovery --source oci:ghcr.io/shashankpai/kairos-pi:v0.1.0
```

No reboot is required for the recovery upgrade — it writes to the recovery partition only.

## Rollback

Kairos has automatic boot assessment: if the newly-upgraded active image fails to boot (kernel panic, systemd crash), the bootloader automatically falls back to the passive image (the one you were running before the upgrade) on the next boot cycle. You usually don't need to do anything.

### Manual rollback

If the active system boots but is broken in a subtler way (services won't start, config wrong):

1. Reboot into the recovery system (interrupt GRUB at boot, choose the recovery entry).
2. From recovery, either:
   - Upgrade the active slot back to the previous known-good tag:
     ```bash
     sudo kairos-agent upgrade --source oci:ghcr.io/shashankpai/kairos-pi:<previous-good-tag>
     ```
   - Or reset the active slot to a specific image:
     ```bash
     sudo kairos-agent reset --source oci:ghcr.io/shashankpai/kairos-pi:<previous-good-tag>
     ```
3. Reboot into the restored active system.

## Verified upgrade record

### v0.1.0 (sha-30630bc) — first custom image upgrade

- **Date:** 2026-08-25
- **Source image:** `ghcr.io/shashankpai/kairos-pi:sha-30630bc`
- **Base image:** `quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1`
- **Pi:** Raspberry Pi 4, USB pen drive ~64 GB, static IP `192.168.1.34`
- **Result:** Success
  - `KAIROS_TEST_VERSION=sha-30630bc3cf2fd1334afd54358c01414322cbf2e9` confirmed in `/etc/os-release`
  - k3s node `Ready`, all system pods running
  - Boot-time config pull (curl+tar) working — `/oem/cloud-config-files/` populated
- **Notes:**
  - Hadron v0.5.1 is a "nogit" build — the `stages.boot.git` module is unavailable. Used `curl+tar` in `stages.boot.commands` instead (see `cloud-config/10_git-pull.yaml`).
  - k3s state was reset before the upgrade due to an IP change (DHCP assigned a new IP after reboot). Set static IP via `/etc/systemd/network/10-end0.network` to prevent recurrence.
  - The upgrade was tested with `:sha-30630bc` (commit-pinned tag). The `:v0.1.0` release tag produces the same image plus a `.raw.gz` flash artifact.

## Upgrade cadence

This plan uses **manual** upgrades (SSH-triggered). A reasonable cadence:

- Track upstream `quay.io/kairos/hadron` releases; when a new hadron tag drops, bump the `FROM` in `image/Dockerfile`, open a PR, let Actions build and push `:latest`.
- Test `:latest` on the Pi by upgrading to `:sha-<short>` first (pinned, reproducible).
- Once verified, tag `vX.Y.Z`, which produces the release `.raw` and the `:vX.Y.Z` image tag.
- Upgrade active, validate, then upgrade recovery.

Promotion paths for later (not implemented now):
- **GitHub Actions SSH**: a workflow that SSHes into the Pi and runs the upgrade on tag push. Centralized and auditable.
- **Kairos Operator**: `NodeOpUpgrade` CRDs applied to the k3s cluster itself. Best for multi-node.
- **Systemd timer on the Pi**: a periodic `kairos-agent upgrade` against `:latest`. Autonomous but less auditable.
