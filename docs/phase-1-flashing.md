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
- A Linux controller machine with Docker and `dd` (we used an Ubuntu machine at 192.168.1.32).
- The Kairos base image tag:
  `quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1`

> **Tag format gotcha:** `kairos-agent upgrade list-releases` displays the image as
> `k3sv1.36.3+k3s1`, but the actual quay.io tag uses hyphens: `k3s-v1.36.3-k3s1`.
> The `+` character is invalid in Docker tags. The produced `.raw` filename uses `+`
> (the `+` comes back in the output filename — this is normal).

---

## Architecture

```
Pi 1 (10.0.0.34) — k3s server + control plane
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

**Hybrid approach:** SSD setup is **baked into the flash-time config** so it runs on first boot
before k3s starts. The repo contains a reconciliation config (`01_ssd_storage.yaml`) that
re-applies the local-path provisioner patch every 60 minutes (drift correction).
- **Boot 1** → SSD detected, formatted, mounted; k3s data is on SSD from the start
- **Every boot** → git-pull syncs repo configs; reconciliation ensures local-path provisioner uses SSD

---

## Step 1: Write the flash-time cloud-config on the controller

The flash-time config is **per-Pi** and is **not committed to this repo**. Create it
directly on the Ubuntu controller machine:

```bash
mkdir -p ~/kairos-flash
cat > ~/kairos-flash/cloud-config-pi1.yaml << 'EOF'
#cloud-config
# Pi 1 flash-time config — k3s server, 10.0.0.34 (VLAN 10)
# NOT committed to the repo (contains SSH keys and passwords).
#
# This config runs ONCE at flash time and sets up:
# 1. DHCP IP (gets 10.0.0.34 from OPNsense Kea, static reservation by MAC)
# 2. SSD detection, formatting, mounting (fs stage — runs FIRST, before k3s)
# 3. k3s data-dir on SSD (so Grafana JS is fast from first boot)
# 4. Git-pull bootstrap (pulls repo on every boot)

users:
  - name: kairos
    groups:
      - admin
    ssh_authorized_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHo5qJ/2w3UcBUoRkXPxkv7nbL2OXEhpopROI906HPBY ansible-controller"

k3s:
  enabled: true

write_files:
  # --- DHCP IP (from OPNsense Kea) ---
  # Pi gets 10.0.0.34 from OPNsense Kea DHCP (static reservation by MAC d8:3a:dd:fc:b7:48).
  # systemd-networkd on this Kairos image always selects 20-dhcp.network
  # for the end0 interface (Pi 4 Gigabit Ethernet).
  # We configure it for DHCP to get the reserved IP from Kea.
  - path: /etc/systemd/network/20-dhcp.network
    permissions: "0644"
    content: |
      [Match]
      Name=end0

      [Network]
      DHCP=yes
      DNS=10.0.0.1
      DNS=8.8.8.8

  # --- SSD mount unit ---
  # Mounts the USB SSD at /usr/local/ssd BEFORE k3s starts.
  # Uses partition label "kairos-ssd" (not UUID) so it works on every Pi.
  # If SSD is not plugged in, the mount fails silently and k3s uses pen drive.
  - path: /etc/systemd/system/usr-local-ssd.mount
    permissions: "0644"
    content: |
      [Unit]
      Description=Mount USB SSD at /usr/local/ssd
      After=local-fs.target
      Before=k3s.service

      [Mount]
      What=LABEL=kairos-ssd
      Where=/usr/local/ssd
      Type=ext4
      Options=defaults,noatime

      [Install]
      WantedBy=multi-user.target

  # --- Git-pull bootstrap ---
  # The ONLY cloud-config file written at flash time (besides this one).
  # On every boot: pulls the repo tarball and syncs cloud-config/*.yaml
  # into /oem/ so all other configs are self-updating from GitHub.
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

stages:
  fs:
    - name: "Detect, format, and mount SSD (first boot only)"
      commands:
        - |
          # Create mount point on the persistent partition (rootfs is read-only)
          mkdir -p /usr/local/ssd /usr/local/ssd/k3s-storage

          # Check if the SSD is already mounted (e.g., from a previous boot)
          if mountpoint -q /usr/local/ssd; then
            exit 0
          fi

          # Check if a partition with label kairos-ssd already exists
          if blkid -L kairos-ssd >/dev/null 2>&1; then
            # SSD is already formatted — just mount it
            systemctl daemon-reload
            systemctl enable usr-local-ssd.mount
            systemctl start usr-local-ssd.mount
            exit 0
          fi

          # SSD is not formatted — find the SSD device.
          # Strategy: find the largest disk that is NOT the boot device.
          BOOT_DISK=""
          # Find which disk has the COS_STATE partition (Kairos boot marker)
          for dev in /dev/sd[a-z] /dev/nvme[0-9]n[0-9]; do
            [ -b "$dev" ] || continue
            if blkid "$dev" 2>/dev/null | grep -q "COS_STATE\|COS_PERSISTENT\|cos-state" || \
               lsblk -no PARTLABEL "$dev" 2>/dev/null | grep -qi "COS"; then
              BOOT_DISK="$dev"
              break
            fi
          done
          # Fallback: the boot disk is the one mounted at /
          if [ -z "$BOOT_DISK" ]; then
            BOOT_DISK=$(findmnt -no SOURCE / | sed 's/[0-9]*$//' | sed 's/p$//')
          fi

          # Find the largest disk that isn't the boot disk
          SSD_DISK=""
          SSD_SIZE=0
          for dev in /dev/sd[a-z] /dev/nvme[0-9]n[0-9]; do
            [ -b "$dev" ] || continue
            [ "$dev" = "$BOOT_DISK" ] && continue
            size=$(blockdev --getsize64 "$dev" 2>/dev/null || echo 0)
            if [ "$size" -gt "$SSD_SIZE" ]; then
              SSD_DISK="$dev"
              SSD_SIZE="$size"
            fi
          done

          if [ -z "$SSD_DISK" ]; then
            echo "No SSD found (only boot disk $BOOT_DISK detected). PVCs will use the boot drive."
            exit 0
          fi

          echo "Found SSD: $SSD_DISK ($((SSD_SIZE / 1024 / 1024 / 1024)) GB), boot disk: $BOOT_DISK"

          # Wipe any existing partition table and create a single partition
          wipefs -a "$SSD_DISK" 2>/dev/null || true
          dd if=/dev/zero of="$SSD_DISK" bs=1M count=10 2>/dev/null || true

          # Create a single partition using fdisk
          echo -e "o\nn\np\n1\n\n\nw" | fdisk "$SSD_DISK" 2>/dev/null || true

          # Wait for the partition device to appear
          sleep 2
          # Handle both /dev/sdX1 and /dev/nvmeXnYp1 naming
          if [ -b "${SSD_DISK}1" ]; then
            SSD_PART="${SSD_DISK}1"
          elif [ -b "${SSD_DISK}p1" ]; then
            SSD_PART="${SSD_DISK}p1"
          else
            echo "Partition not found after fdisk. Trying whole disk."
            SSD_PART="$SSD_DISK"
          fi

          # Format with ext4 and the kairos-ssd label
          mkfs.ext4 -F -L kairos-ssd "$SSD_PART"

          # Mount via systemd
          systemctl daemon-reload
          systemctl enable usr-local-ssd.mount
          systemctl start usr-local-ssd.mount

          echo "SSD formatted and mounted at /usr/local/ssd"

          # Configure k3s to use the SSD for ALL its data (container image layers,
          # etcd, TLS certs, PVC storage). This is the key performance fix:
          # containerd's overlay filesystem for container images lives here, so
          # static files (JS/CSS) are served from fast SSD I/O, not the pen drive.
          mkdir -p /etc/rancher/k3s /usr/local/ssd/k3s-data
          if ! grep -q "data-dir" /etc/rancher/k3s/config.yaml 2>/dev/null; then
            cat >> /etc/rancher/k3s/config.yaml << 'EOCONF'
data-dir: /usr/local/ssd/k3s-data
EOCONF
          fi
EOF
```

Verify it looks right:
```bash
cat ~/kairos-flash/cloud-config-pi1.yaml
```

### What this flash-time config does

| Stage | What | Why |
|-------|------|-----|
| **fs** (first) | Detect SSD, format with `kairos-ssd` label, mount at `/usr/local/ssd` | Runs BEFORE k3s starts, so k3s data is on SSD from first boot |
| **fs** (first) | Write k3s config: `data-dir: /usr/local/ssd/k3s-data` | All k3s data (container images, etcd, PVCs) goes to SSD, not pen drive |
| **boot** | Pull repo tarball, sync `cloud-config/*.yaml` to `/oem/` | Makes cloud-config self-updating from GitHub |

### Why this matters

- **First boot:** SSD is mounted and formatted before k3s starts. k3s immediately uses SSD for all data.
- **Grafana performance:** Container image layers (including Grafana's JS files) are on SSD. JS files load in ~30ms instead of ~30s.
- **Fleet consistency:** Every Pi uses the same flash-time config, so SSD setup is identical across the fleet.

---

## Step 2: Build the flash image with AuroraBoot

AuroraBoot pulls the Kairos container image and produces a bootable `.raw` with the
cloud-config injected. Runs on the controller (needs Docker).

```bash
KAIROS_IMAGE=quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1

sudo docker run --rm --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/kairos-flash:/output" \
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

Takes 3–5 minutes (pulls ~1GB image). Success looks like:
```
INF Assembled final disk image target=/output/kairos-hadron-v0.5.1-standard-arm64-rpi4-v4.2.0-k3sv1.36.3+k3s1.raw
```

Output file: `~/kairos-flash/kairos-hadron-v0.5.1-standard-arm64-rpi4-v4.2.0-k3sv1.36.3+k3s1.raw`

---

## Step 3: Identify the USB pen drive

Plug the USB pen drive into the controller and check which device it is:

```bash
lsblk -d -o NAME,SIZE,MODEL,TRAN
```

Look for the USB entry (TRAN=usb). On our setup:
```
NAME      SIZE MODEL              TRAN
sda      57.7G Sandisk 3.2Gen1   usb    ← this is the pen drive
nvme0n1 476.9G INTEL SSDPEKNW... nvme   ← DO NOT TOUCH (controller's own SSD)
```

---

## Step 4: Flash to the USB pen drive

**This destroys all data on the target device — double-check the device name.**

```bash
# Unmount any partitions that auto-mounted
sudo umount /dev/sda* 2>/dev/null || true

# Flash (5-10 minutes)
sudo dd \
  if=~/kairos-flash/kairos-hadron-v0.5.1-standard-arm64-rpi4-v4.2.0-k3sv1.36.3+k3s1.raw \
  of=/dev/sda \
  bs=4M \
  status=progress \
  conv=fsync

# Flush and eject
sync
sudo eject /dev/sda
```

---

## Step 5: Boot Pi 1

1. Unplug the USB pen drive from the controller.
2. Plug both the **USB pen drive** and the **USB SSD** into Pi 1.
3. Power on Pi 1.

**First boot** creates the `COS_STATE` and `COS_PERSISTENT` partitions and reboots
automatically once (~2 min). After the automatic reboot, SSH in:

```bash
ssh -i ~/.ssh/id_ed25519 kairos@192.168.1.34
```

> If you can't reach 192.168.1.34, check your router's DHCP leases — the Pi should show
> MAC `d8:3a:dd:fc:b7:48`. If static IP failed, see the troubleshooting section below.

---

## Step 6: Verify Boot 1

```bash
# Static IP is set (should show 192.168.1.34, NOT dynamic)
ip addr show end0 | grep inet

# Which network files are active
sudo networkctl status end0 | grep -E "Network File|Address"

# k3s is running
sudo systemctl status k3s --no-pager | head -8

# Git-pull synced the repo (check /oem/ for 05_ssd_storage.yaml)
ls /oem/*.yaml
```

At this point `05_ssd_storage.yaml` should be in `/oem/` (synced by the git-pull boot stage).
The SSD is not yet mounted — it will be on the next reboot.

---

## Step 7: Second reboot — activate SSD

```bash
sudo reboot
```

Wait ~2 minutes, then SSH back to 192.168.1.34. Verify:

```bash
# SSD is mounted
mountpoint -q /usr/local/ssd && echo "SSD mounted" || echo "NOT mounted"

# k3s is using SSD (data-dir should point to SSD)
sudo cat /etc/rancher/k3s/config.yaml

# k3s data is on the SSD
ls /usr/local/ssd/k3s-data/

# Allow 5-10 min for image pulls on first SSD boot, then check pods
sudo k3s kubectl get pods -n monitoring
```

Expected state after Boot 2:
- SSD mounted at `/usr/local/ssd`
- k3s config: `data-dir: /usr/local/ssd/k3s-data`
- Monitoring pods coming up (image pulls to SSD take a few minutes)
- Grafana at `http://192.168.1.34:3000` loads in < 2s

---

## Step 8: Verify Grafana performance

```bash
# Test JS file load time from the Pi itself (should be < 100ms)
curl -s -o /dev/null -w "%{time_total}s\n" http://localhost:3000/public/build/app.*.js 2>/dev/null \
  || curl -s -o /dev/null -w "%{time_total}s\n" http://localhost:3000/login
```

From a browser: open `http://192.168.1.34:3000` — the login page should appear in under 2s.
Previously on pen drive this took 30s+.

---

## What you have at the end of Phase 1

- **Pi 1** at 192.168.1.34: k3s server, static IP, SSD-backed k3s data
- Self-updating cloud-config from GitHub on every boot (push to repo → reboots apply)
- Monitoring stack (Grafana, Prometheus, node-exporter, kube-state-metrics) deployed

---

## Troubleshooting

### Static IP not applied — Pi comes up with a different IP

Check which network file networkd is actually using:

```bash
sudo networkctl status end0 | grep -E "Network File|Address"
sudo cat /etc/systemd/network/20-dhcp.network
```

`20-dhcp.network` should show `Name=end0` and `DHCP=no`. If it still shows the stock
`Name=en*` / `DHCP=yes`, the `write_files` didn't take effect. Fix it now:

```bash
sudo bash -c 'cat > /etc/systemd/network/20-dhcp.network << EOF
[Match]
Name=end0

[Network]
DHCP=no
Address=192.168.1.34/24
Gateway=192.168.1.1
DNS=192.168.1.1
DNS=8.8.8.8
EOF'
sudo networkctl reload && sudo networkctl reconfigure end0
sleep 3
ip addr show end0 | grep inet
```

> **Why this approach?** On the Kairos Hadron v0.5.1 (systemd-networkd on musl), only
> `20-dhcp.network` is ever applied to `end0`, regardless of other matching files or sort
> order. The `!end0` negation syntax is also not supported. Overwriting `20-dhcp.network`
> directly is the only reliable fix.

### SSD not detected after Boot 2

```bash
lsblk                                         # check SSD is visible (sdb or similar)
blkid -L kairos-ssd                           # check if label is set
sudo systemctl status usr-local-ssd.mount     # check mount unit
sudo journalctl -u usr-local-ssd.mount --no-pager | tail -20
```

If the SSD has no `kairos-ssd` label: check that `05_ssd_storage.yaml` is in `/oem/` and
reboot again. The `fs` stage will detect, format, and mount it.

### k3s not starting after Boot 2

```bash
sudo journalctl -u k3s.service --no-pager | grep -i "error\|fatal\|panic" | tail -20
# If SSD isn't mounted but k3s config says data-dir=/usr/local/ssd/..., k3s fails.
mountpoint -q /usr/local/ssd && echo "SSD mounted" || echo "Mount failed — reboot"
```

### k3s kubectl permission denied

```bash
sudo k3s kubectl get nodes    # always use sudo on Kairos
```

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
