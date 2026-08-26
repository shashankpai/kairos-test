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

**Two-boot sequence:** The SSD config is self-updating from git (not baked into the image).
So it takes two boots to fully activate the SSD:
- **Boot 1** → static IP .34 is live, git-pull syncs `05_ssd_storage.yaml` into `/oem/`
- **Boot 2** (manual reboot) → SSD detected, formatted, mounted, k3s uses SSD from here

---

## Step 1: Write the flash-time cloud-config on the controller

The flash-time config is **per-Pi** and is **not committed to this repo**. Create it
directly on the Ubuntu controller machine:

```bash
mkdir -p ~/kairos-flash
cat > ~/kairos-flash/cloud-config-pi1.yaml << 'EOF'
#cloud-config
users:
  - name: kairos
    groups:
      - admin
    ssh_authorized_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHo5qJ/2w3UcBUoRkXPxkv7nbL2OXEhpopROI906HPBY ansible-controller"

k3s:
  enabled: true

write_files:
  # --- Static IP fix ---
  # Overwrite 20-dhcp.network to NOT match end0 (Pi 4 Ethernet).
  # systemd-networkd merges all matching .network files with later filenames
  # overriding earlier ones. By excluding end0 from the DHCP config, our
  # 99-end0-static.network below becomes the sole match for end0.
  - path: /etc/systemd/network/20-dhcp.network
    permissions: "0644"
    content: |
      [Match]
      Name=en* !end0

      [Network]
      DHCP=yes
      [DHCP]
      ClientIdentifier=mac

  # Static IP for Pi 1. end0 is the Pi 4's built-in Gigabit Ethernet.
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
  # The ONLY cloud-config file written at flash time.
  # On every boot: pulls the repo tarball, syncs cloud-config/*.yaml into /oem/.
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
EOF
```

Verify it looks right:
```bash
cat ~/kairos-flash/cloud-config-pi1.yaml
```

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

Check the network config files and which one networkd is actually using:

```bash
sudo ls -la /etc/systemd/network/
sudo networkctl status end0 | grep "Network File"
sudo cat /etc/systemd/network/20-dhcp.network     # should say Name=en* !end0
sudo cat /etc/systemd/network/99-end0-static.network
```

If `20-dhcp.network` still says `Name=en*` (without `!end0`), the `write_files` from the
flash-time config didn't take effect. This can happen if Kairos's own init re-generated the
file after ours was written. Force it now:

```bash
sudo bash -c 'cat > /etc/systemd/network/20-dhcp.network << EOF
[Match]
Name=en* !end0

[Network]
DHCP=yes
[DHCP]
ClientIdentifier=mac
EOF'
sudo networkctl reload && sudo networkctl reconfigure end0
ip addr show end0 | grep inet
```

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
