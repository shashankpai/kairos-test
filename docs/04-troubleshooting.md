# Troubleshooting

Common failures and how to diagnose them on the Pi.

## First stop: kairos-agent logs

```bash
ssh kairos@<pi>
sudo journalctl -fu kairos-agent
```

This service runs at first boot to set up k3s/edgevpn, and on every boot to apply cloud-config stages. Most config errors show up here.

## Cloud-config stage errors

If a `stages.boot.*` entry fails, the boot continues but the stage is logged as failed. Check:

```bash
sudo journalctl -b | grep -i 'cloud-init\|kairos\|stage'
```

Common causes:
- YAML syntax error — validate locally with `yq` or `python -c 'import yaml; yaml.safe_load(open("f.yaml"))'`.
- **`git plugin not available in nogit build`** — Hadron v0.5.1 is a "nogit" build. The `stages.boot.git` module is not available, and `git` is not installed. Use `stages.boot.commands` with `curl` + `tar` instead (see `cloud-config/10_git-pull.yaml` for the working pattern).
- `commands:` referencing a binary not in the image — hadron doesn't ship `apk`, so you can't install packages at runtime. Add it to `image/Dockerfile` and rebuild, or use a different approach (e.g., `curl` instead of `git`).

## k3s not starting

```bash
sudo systemctl status k3s
sudo journalctl -fu k3s
sudo k3s kubectl get nodes    # k3s embeds kubectl; kubeconfig is root-only
```

The `kairos-agent` first-boot service writes `/etc/sysconfig/k3s`. If it didn't run (check `journalctl -fu kairos-agent` for the first boot), k3s won't be configured. Removing `/usr/local/.kairos/deployed` forces the agent to re-run on next boot:

```bash
sudo rm /usr/local/.kairos/deployed
sudo reboot
```

## Can't SSH in

- Verify the Pi is on the network: `ping <pi>`, check router DHCP leases.
- The flash-time cloud-config must have your SSH key in `users[*].ssh_authorized_keys`. If you forgot it, you'll need to re-flash or attach a monitor + keyboard and edit `/oem/90_custom.yaml` from a recovery boot.
- If you used a password and it's not working, recovery boot and check `/oem/90_custom.yaml`.

## Boot loop after upgrade

Kairos should auto-fall-back to the passive image after a few failed boots. If it doesn't, or if you want to force it:

1. Interrupt GRUB at boot (hold a key during the menu).
2. Choose the passive or recovery entry.
3. From recovery, roll back as described in [02-upgrades.md](02-upgrades.md#manual-rollback).

## Inspecting partitions from recovery

From the recovery system, the active/passive state partition is mounted under `/run/initramfs/cos-state`:

```bash
ls /run/initramfs/cos-state/cOS/
# active.img  passive.img  ...
```

## USB pen drive wear / endurance

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

If you see repeated corruption, re-flash the pen drive and restore from a known-good image tag (see [02-upgrades.md](02-upgrades.md#manual-rollback)).

## Useful files on the Pi

| Path | Contents |
| --- | --- |
| `/oem/90_custom.yaml` | The flash-time cloud-config (written by AuroraBoot). |
| `/oem/cloud-config-files/` | Repo contents pulled at boot (Phase 2). |
| `/usr/local/.kairos/deployed` | Sentinel; presence means first-boot setup is done. |
| `/etc/sysconfig/k3s` | k3s config written by kairos-agent. |
| `/run/initramfs/cos-state/cOS/` | Active/passive OS images (recovery only). |
