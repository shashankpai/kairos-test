# Phase 1: Flashing and booting the Kairos Pi

How the Raspberry Pi 4 was initially provisioned — from a stock Kairos hadron image to a bootable USB pen drive running k3s. This is the one-time flashing procedure. All subsequent updates happen via GitHub-managed upgrades (see [phase-2-github-managed.md](phase-2-github-managed.md)).

## Goal

Get a Raspberry Pi 4 booting Kairos hadron from a USB pen drive, with k3s running and SSH access via a key. No GitHub integration yet — just a working immutable OS node.

## Prerequisites

- Raspberry Pi 4 (arm64).
- USB pen drive (~64 GB) to boot from. An SD card also works, but a pen drive has better endurance for k3s writes.
- A Linux/Mac host with Docker and `dd`.
- The Kairos base image tag. We used:
  `quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1`

> **Tag format gotcha:** `kairos-agent upgrade list-releases` displays the image as `k3sv1.36.3+k3s1`, but the actual pullable quay.io tag uses hyphens: `k3s-v1.36.3-k3s1`. The `+` character is invalid in Docker tags. Always verify the tag against the quay.io API or web UI before using it in a `FROM` or `docker pull`.

## Step 1: Build the flash image with AuroraBoot

AuroraBoot wraps the upstream Kairos container image into a bootable `.raw` and injects a cloud-config.

Create a local working directory (this is the gitignored `build/` dir; do **not** commit its contents):

```bash
mkdir -p build
```

Write the **flash-time** cloud-config to `build/cloud-config.yaml`. At minimum this needs a user, SSH key, and k3s enabled:

```yaml
#cloud-config
users:
  - name: kairos
    passwd: "Kairo@987"          # flash-time only; NOT committed to the repo
    groups:
      - admin
    ssh_authorized_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHo5qJ/2w3UcBUoRkXPxkv7nbL2OXEhpopROI906HPBY ansible-controller"
k3s:
  enabled: true
```

Run AuroraBoot to produce the `.raw`:

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

## Step 2: Flash to the USB pen drive

Identify the device (`diskutil list` on macOS, `lsblk` on Linux). **This destroys all data on the target device — verify the device path carefully.**

```bash
# On Linux:
sudo dd \
  if=build/kairos-hadron-v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1.raw \
  of=/dev/sdX \
  bs=4M \
  status=progress \
  conv=fsync

# On macOS, use /dev/rdiskN (raw disk, faster) and unmount first:
diskutil unmountDisk /dev/diskN
sudo dd \
  if=build/kairos-hadron-v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1.raw \
  of=/dev/rdiskN \
  bs=4m \
  status=progress
```

## Step 3: Boot the Pi

1. Plug the USB pen drive into the Pi 4.
2. Power on. On first boot, Kairos creates the `COS_STATE` and `COS_PERSISTENT` volumes and reboots once automatically.
3. After the automatic reboot, the Pi should be on the network via DHCP.

## Step 4: Verify the node

Find the Pi's IP (check your router's DHCP leases, or use `nmap`/`arp`). SSH in:

```bash
ssh -i ~/.ssh/id_ed25519 kairos@<pi-ip>
```

Verify the OS and k3s:

```bash
# Check the image version
sudo kairos-agent upgrade list-releases

# Check k3s (kubectl is embedded in k3s; kubeconfig is root-only)
sudo k3s kubectl get nodes -o wide
```

Expected output:
```
NAME          STATUS   ROLES           AGE   VERSION        INTERNAL-IP    OS-IMAGE       KERNEL-VERSION         CONTAINER-RUNTIME
kairos-XXXX   Ready    control-plane   Xm    v1.36.3+k3s1   192.168.1.X    Hadron Linux   7.1.3-hadron (arm64)   containerd://2.3.2-k3s2
```

Verify the disk and partitions:

```bash
lsblk
# Should show the USB pen drive with COS_STATE, COS_PERSISTENT, and OEM partitions
```

## What you have at the end of Phase 1

- A Raspberry Pi 4 booting Kairos hadron v0.5.1 from a USB pen drive.
- k3s running as a single-node control plane.
- SSH access via the `kairos` user with your ed25519 key.
- No GitHub integration, no custom image, no boot-time config pull.
- The `/oem/` directory contains only `90_custom.yaml` (the flash-time cloud-config) and `grubenv`.

## Troubleshooting encountered during Phase 1

### k3s not starting after first boot

The `kairos-agent` first-boot service writes `/etc/sysconfig/k3s`. If it didn't run (check `journalctl -fu kairos-agent` for the first boot), k3s won't be configured.

**Fix:** Remove the first-boot sentinel to force the agent to re-run:

```bash
sudo rm /usr/local/.kairos/deployed
sudo reboot
```

### Can't SSH in

- Verify the Pi is on the network: `ping <pi>`, check router DHCP leases.
- The flash-time cloud-config must have your SSH key in `users[*].ssh_authorized_keys`. If you forgot it, you'll need to re-flash or attach a monitor + keyboard and edit `/oem/90_custom.yaml` from a recovery boot.
- If you used a password and it's not working, recovery boot and check `/oem/90_custom.yaml`.

### k3s kubectl permission denied

On Kairos, the kubeconfig is root-only. You must use `sudo k3s kubectl` instead of plain `kubectl`:

```bash
# Wrong:
kubectl get nodes                    # permission denied

# Right:
sudo k3s kubectl get nodes           # works
```

### USB pen drive wear / endurance

This node boots from a ~64 GB USB pen drive rather than an SSD. Pen drives typically have lower write endurance (fewer erase cycles per cell) and slower random I/O than SSDs. Most of the Kairos rootfs is read-only (immutable), so the OS itself isn't a wear concern. The risk is the **persistent partition** (`/usr/local` and `/oem`), which k3s writes to — k3s stores its state in a sqlite-backed etcd (`/var/lib/rancher/k3s/server/db/`), and frequent small writes can wear out cheaper flash over months of uptime.

Symptoms of flash wear:
- k3s becomes unstable, nodes flap, or `kubectl get nodes` intermittently fails.
- Filesystem errors in `dmesg` (I/O errors, remounted read-only).
- Slow boot or upgrade times that progressively worsen.

Mitigations, in order of effort:
- Move k3s state to an external SSD or a separate, higher-endurance USB device (bind-mount `/var/lib/rancher/k3s`).
- Use `k3s`'s embedded etcd with a WAL on a more durable medium, or switch the node to a k3s **agent** pointing at a server elsewhere (no local etcd).
- Reduce k3s write frequency (e.g. disable unused controllers, lower `--kube-controller-manager` sync periods) — marginal gains.
- Replace the pen drive with a USB SSD or a higher-endurance industrial SD card + USB adapter.

### Boot loop after upgrade

Kairos should auto-fall-back to the passive image after a few failed boots. If it doesn't, or if you want to force it:

1. Interrupt GRUB at boot (hold a key during the menu).
2. Choose the passive or recovery entry.
3. From recovery, roll back as described in [phase-2-github-managed.md](phase-2-github-managed.md#manual-rollback).

### Inspecting partitions from recovery

From the recovery system, the active/passive state partition is mounted under `/run/initramfs/cos-state`:

```bash
ls /run/initramfs/cos-state/cOS/
# active.img  passive.img  ...
```
