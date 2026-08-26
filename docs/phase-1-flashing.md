# Phase 1: Flashing and booting the Kairos Pi

How to provision a Raspberry Pi 4 from scratch — from a stock Kairos hadron image to a
running k3s node with a static IP, SSD-backed storage, and self-updating config from GitHub.

This document covers Pi 1 (k3s server). Pi 2 (agent) will be added in a later phase.

---

## Prerequisites

- Raspberry Pi 4 (arm64).
- USB pen drive (~64 GB) to boot from.
- USB SSD connected to the Pi. Auto-detected, formatted, and mounted by
  `cloud-config/05_ssd_storage.yaml` on first boot.
- A Mac (or Linux) host with Docker and `dd`.
- The Kairos base image tag:
  `quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1`

> **Tag format gotcha:** `kairos-agent upgrade list-releases` displays the image as
> `k3sv1.36.3+k3s1`, but the actual quay.io tag uses hyphens: `k3s-v1.36.3-k3s1`.
> The `+` character is invalid in Docker tags.

---

## Architecture

```
Pi 1 (192.168.1.34) — k3s server + control plane
  ├── USB pen drive → Kairos OS (immutable, read-only rootfs)
  └── USB SSD → k3s data (containerd images, etcd, PVCs)
       /usr/local/ssd/k3s-data/       ← all k3s data (containerd layers live here!)
       /usr/local/ssd/k3s-storage/    ← PVC storage (Prometheus data, Grafana DB)
```

**Why the SSD for k3s data?** Grafana's JS files and all container image layers are served
from containerd's overlay filesystem. On a USB pen drive, even a single 87KB file takes 30s
to read. On an SSD, the same file reads in milliseconds. The `data-dir` setting moves ALL
k3s data (containerd, etcd, TLS certs, PVCs) to the SSD from first boot — no migration ever
needed.

---

## Step 1: Write the flash-time cloud-config

The flash-time config is **per-Pi** and is **not committed to this repo** (it contains a
password). Write it to `build/cloud-config-pi1.yaml` (gitignored).

### Pi 1 — k3s server, 192.168.1.34

```bash
mkdir -p build
```

```yaml
# build/cloud-config-pi1.yaml — Pi 1 (k3s server)
# This file is PER-PI and NOT committed to the repo.
# It contains: user/SSH, static IP, k3s server, and git-pull bootstrap.
# Everything else (SSD storage, k8s workloads) comes from the git repo
# via 10_git-pull.yaml on subsequent boots.
#
#cloud-config
users:
  - name: kairos
    passwd: "Kairo@987"
    groups:
      - admin
    ssh_authorized_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHo5qJ/2w3UcBUoRkXPxkv7nbL2OXEhpopROI906HPBY ansible-controller"

k3s:
  enabled: true

write_files:
  # --- Static IP ---
  # Override 20-dhcp.network to NOT match end0 (the Pi 4 Ethernet interface).
  # systemd-networkd merges all matching .network files (later names override
  # earlier ones for conflicting keys). By making 20-dhcp.network exclude end0,
  # our static config below is the sole match for end0.
  - path: /etc/systemd/network/20-dhcp.network
    permissions: "0644"
    content: |
      [Match]
      Name=en* !end0

      [Network]
      DHCP=yes
      [DHCP]
      ClientIdentifier=mac

  # Static IP for Pi 1. Name=end0 is the Pi 4's built-in Gigabit Ethernet.
  - path: /etc/systemd/network/99-end0-static.network
    permissions: "0644"
    content: |
      [Match]
      Name=end0

      [Network]
      DHCP=no
      Address=192.168.1.34/24
      Gateway=192.168.1.1
      DNS=192.168.1.1
      DNS=8.8.8.8

  # --- Git-pull bootstrap ---
  # This is the ONLY cloud-config file that must be written at flash time.
  # On every boot it pulls the repo and syncs cloud-config/*.yaml into /oem/,
  # making all other config self-updating via GitHub.
  - path: /oem/10_git-pull.yaml
    permissions: "0644"
    content: |
      #cloud-config
      stages:
        boot:
          - name: "Pull config repo and sync cloud-config"
            commands:
              - |
                set -e
                mkdir -p /oem/cloud-config-files
                tmpdir=$(mktemp -d)
                curl -sL "https://github.com/shashankpai/kairos-test/archive/refs/heads/main.tar.gz" -o "$tmpdir/repo.tar.gz"
                rm -rf /oem/cloud-config-files/*
                tar xzf "$tmpdir/repo.tar.gz" -C /oem/cloud-config-files/ --strip-components=1
                rm -rf "$tmpdir"
                if [ -d /oem/cloud-config-files/cloud-config ]; then
                  for f in /oem/cloud-config-files/cloud-config/*.yaml; do
                    [ -f "$f" ] && cp "$f" /oem/
                  done
                fi
```

> **Note:** The `write_files` for `20-dhcp.network` replaces the stock Kairos DHCP config
> with one that excludes `end0`. Without this, systemd-networkd applies both DHCP and static
> configs to `end0`, with the DHCP config winning because `20-dhcp.network` is applied after
> our `99-end0-static.network` alphabetically.

---

## Step 2: Build the flash image with AuroraBoot

AuroraBoot wraps the upstream Kairos container image into a bootable `.raw` and injects the
cloud-config.

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
  --cloud-config /output/cloud-config-pi1.yaml \
  --set "container_image=$KAIROS_IMAGE"
```

Output: `build/kairos-hadron-*.raw`

---

## Step 3: Flash to the USB pen drive

Identify the device (`diskutil list` on macOS, `lsblk` on Linux).
**This destroys all data on the target device — verify the device path carefully.**

```bash
# On macOS (use /dev/rdiskN for raw disk, unmount first):
diskutil unmountDisk /dev/diskN
sudo dd \
  if=build/kairos-hadron-*.raw \
  of=/dev/rdiskN \
  bs=4m

# On Linux:
sudo dd \
  if=build/kairos-hadron-*.raw \
  of=/dev/sdX \
  bs=4M \
  status=progress \
  conv=fsync
```

---

## Step 4: Boot Pi 1

1. Plug the flashed USB pen drive and the USB SSD into the Pi 4.
2. Power on. On first boot, Kairos creates the `COS_STATE` and `COS_PERSISTENT` partitions
   and reboots automatically.
3. After the reboot, SSH in at **192.168.1.34**:

```bash
ssh -i ~/.ssh/id_ed25519 kairos@192.168.1.34
```

Verify:
```bash
# Static IP is set correctly
ip addr show end0 | grep inet

# k3s is running
sudo systemctl status k3s | head -5

# SSD is mounted
mountpoint -q /usr/local/ssd && echo "SSD mounted" || echo "SSD not mounted"

# k3s data is on the SSD (05_ssd_storage.yaml sets this up)
ls /usr/local/ssd/k3s-data/ 2>/dev/null || echo "Waiting for first git-pull boot..."

# All monitoring pods eventually come up (allow 3-5 min for image pulls)
sudo k3s kubectl get pods -n monitoring
```

> **First boot takes 3–5 minutes** for image pulls. The git-pull runs at the `boot` stage,
> syncing `05_ssd_storage.yaml` and `20_k8s_workloads.yaml` into `/oem/`. On the SECOND
> boot, the SSD data-dir config is applied and k3s uses the SSD for all data.

## What you have at the end of Phase 1

- **Pi 1**: k3s server at 192.168.1.34, static IP, SSD-backed k3s data
- Self-updating cloud-config from GitHub on every boot
- Monitoring stack (Grafana, Prometheus, node-exporter, kube-state-metrics) deployed

---

## Troubleshooting

### Static IP not applied after first boot

The `20-dhcp.network` override in the flash-time config prevents networkd from assigning a
DHCP address to `end0`. If the IP is still dynamic after first boot:

```bash
# Check which network file is active
sudo networkctl status end0

# Verify our files are in place
sudo ls -la /etc/systemd/network/
sudo cat /etc/systemd/network/99-end0-static.network

# Force reconfigure
sudo networkctl reload && sudo networkctl reconfigure end0
```

### k3s not starting after first boot

```bash
sudo journalctl -u k3s.service --no-pager | grep -i "error\|fatal" | tail -20
# If data-dir is on SSD and SSD isn't mounted yet, k3s fails.
# Check SSD mount: mountpoint -q /usr/local/ssd && echo mounted
# The usr-local-ssd.mount service must start before k3s:
sudo systemctl status usr-local-ssd.mount
```

### k3s kubectl permission denied

```bash
# Always use sudo on Kairos
sudo k3s kubectl get nodes
```

### Images pulling slowly on first boot

Container images are pulled directly to the SSD (`/usr/local/ssd/k3s-data/agent/containerd/`).
First pull takes a few minutes depending on your network. Subsequent boots are instant
(images are cached on SSD).

### Boot loop after upgrade

Kairos auto-falls-back to the passive image after a few failed boots. To force rollback:
1. Interrupt GRUB at boot.
2. Choose the passive entry.
3. From recovery: `sudo kairos-agent rollback`

### Inspecting partitions from recovery

```bash
ls /run/initramfs/cos-state/cOS/
# active.img  passive.img  ...
```
