# Flashing a node

How the Raspberry Pi 4 was initially provisioned. This is a one-time procedure per node; subsequent updates happen via [upgrades](02-upgrades.md) and [config pulls](03-config-management.md), not re-flashing.

## Prerequisites

- Raspberry Pi 4 (arm64).
- USB pen drive (~64 GB) to boot from (an SD card also works).
- A Linux/Mac host with Docker and `dd`.
- The Kairos base image tag you want to flash. Current:
  `quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1`

## 1. Build the flash image with AuroraBoot

AuroraBoot wraps the upstream Kairos container image into a bootable `.raw` and injects a cloud-config.

Create a local working directory (this is the gitignored `build/` dir; do **not** commit its contents):

```bash
mkdir -p build
```

Write the **flash-time** cloud-config to `build/cloud-config.yaml`. This file is the union of the canonical `cloud-config/00_base.yaml` (from this repo) plus the boot-time git-pull stage from `cloud-config/10_git-pull.yaml`, plus any flash-time-only secrets (initial password, deploy key if the repo is private).

Example flash-time `build/cloud-config.yaml` (public-repo variant — no deploy key):

```yaml
#cloud-config
users:
  - name: kairos
    passwd: "Kairo@987"          # flash-time only; NOT committed
    groups:
      - admin
    ssh_authorized_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHo5qJ/2w3UcBUoRkXPxkv7nbL2OXEhpopROI906HPBY ansible-controller"
k3s:
  enabled: true
stages:
  boot:
    - name: "Pull config repo"
      commands:
        - |
          set -e
          mkdir -p /oem/cloud-config-files
          tmpdir=$(mktemp -d)
          curl -sL "https://github.com/shashankpai/kairos-test/archive/refs/heads/main.tar.gz" -o "$tmpdir/repo.tar.gz"
          rm -rf /oem/cloud-config-files/*
          tar xzf "$tmpdir/repo.tar.gz" -C /oem/cloud-config-files/ --strip-components=1
          rm -rf "$tmpdir"
```

> If the repo is **private**, embed a GitHub personal access token in the URL: `https://<token>@github.com/...`. The token must never be committed — keep it in the flash-time config only.

Run AuroraBoot:

```bash
KAIROS_IMAGE=quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1

sudo docker run --rm --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$PWD/build:/output" \
  quay.io/kairos/auroraboot:latest \
  --debug \
  --set "disable_http_server=true" \
  --set "disable_netboot=true" \
  --set "state_dir=/output" \
  --set "disk.raw=true" \
  --set "arch=arm64" \
  --cloud-config /output/cloud-config.yaml \
  --set "container_image=$KAIROS_IMAGE"
```

Output: `build/kairos-hadron-v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1.raw`.

## 2. Flash to the SSD

Identify the device (`diskutil list` on macOS, `lsblk` on Linux). **This destroys all data on the target device — verify the device path carefully.**

```bash
sudo dd \
  if=~/kairos-pi/build/kairos-hadron-v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1.raw \
  of=/dev/sdd \
  bs=4M \
  status=progress \
  conv=fsync
```

## 3. Boot

Plug the SSD into the Pi and power on. On first boot Kairos creates the `COS_STATE` and `COS_PERSISTENT` volumes and reboots once. After that:

- SSH in as `kairos` using the `ansible-controller` key.
- `sudo kairos-agent upgrade list-releases` should show the current image.
- `kubectl get nodes` should show the node Ready (k3s started by the `kairos-agent` first-boot service).

## 4. Verify the boot-time git pull

After boot:

```bash
ssh kairos@<pi>
ls /oem/cloud-config-files/   # should contain the repo's cloud-config/ contents
```

If empty, check `journalctl -fu kairos-agent` and the `stages.boot.git` entry — most common cause is private-repo auth (deploy key missing or wrong).

## Secrets

- `build/cloud-config.yaml` is gitignored. It contains the initial password and (if private) the deploy key. Keep it local only.
- The canonical `cloud-config/*.yaml` files in the repo must be secret-free. After the first boot, the Pi pulls those files and they override/augment the flash-time config; the password in `/oem/90_custom.yaml` (written at flash time) can be removed in a later config change once SSH key access is confirmed.
