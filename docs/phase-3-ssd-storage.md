# Phase 3: SSD-backed k3s storage

How and why all k3s data (container images, etcd, PVCs) is stored on a USB SSD rather than
the boot USB pen drive — and what that means for performance and operations.

---

## Why the SSD matters for Grafana (and everything else)

Grafana's UI is a React app. When the browser opens `http://<pi>:3000`, it downloads ~1.5MB
of JavaScript files. These files live in Grafana's container image layers, which are stored in
containerd's overlay filesystem on whatever disk holds the k3s data directory.

**On a USB pen drive (pen drive is the boot OS disk):**
- 87KB JS file: ~30 seconds to serve
- Browser makes 6 parallel requests → most time out → blank page

**On a USB SSD (k3s data-dir on SSD):**
- 87KB JS file: ~30ms to serve (1000× faster)
- All 6 requests complete in < 1 second → login page appears instantly

The root cause: k3s's default `data-dir` is `/var/lib/rancher/k3s`, which lives on
`sda5` (the COS_PERSISTENT partition on the pen drive). Random reads from a cheap USB
pen drive are orders of magnitude slower than an SSD.

---

## What lives on the SSD

```
/usr/local/ssd/
├── k3s-data/                  # k3s --data-dir (set in /etc/rancher/k3s/config.yaml)
│   ├── agent/
│   │   └── containerd/        # container image layers (Grafana JS files live HERE)
│   ├── data/                  # k3s binary assets
│   └── server/
│       ├── db/                # etcd / sqlite WAL
│       └── tls/               # TLS certificates
└── k3s-storage/               # local-path provisioner PVC storage
    └── monitoring/
        ├── prometheus-pvc/    # Prometheus TSDB (time series data)
        └── grafana-pvc/       # Grafana SQLite + dashboards
```

The pen drive holds only the immutable Kairos OS — squashfs + GRUB. No workload I/O
touches it during normal operation.

---

## How it's configured

### `/etc/rancher/k3s/config.yaml` (written by `05_ssd_storage.yaml`)

```yaml
data-dir: /usr/local/ssd/k3s-data
```

k3s reads this file on startup. When the SSD is mounted at `/usr/local/ssd` before k3s
starts, k3s uses the SSD for all its data from first boot.

### `05_ssd_storage.yaml` — what it does on each boot

1. **`fs` stage** — very early, before k3s starts:
   - Creates `/usr/local/ssd` mount point
   - If already mounted: no-op
   - If SSD has `LABEL=kairos-ssd`: enables and starts `usr-local-ssd.mount`
   - If SSD is raw/unformatted: detects the largest non-boot disk, formats it
     with `mkfs.ext4 -L kairos-ssd`, mounts it, writes k3s `data-dir` config

2. **`network` stage** — after k3s is ready:
   - Patches the local-path provisioner ConfigMap to store PVCs in
     `/usr/local/ssd/k3s-storage`

3. **`reconcile` stage** — every 60 minutes:
   - Re-applies the local-path provisioner patch (drift correction)

### `usr-local-ssd.mount` (written by `05_ssd_storage.yaml`)

```ini
[Mount]
What=LABEL=kairos-ssd
Where=/usr/local/ssd
Type=ext4
Options=defaults,noatime

[Unit]
Before=k3s.service
```

`Before=k3s.service` ensures the SSD is mounted before k3s starts. If the SSD is absent
(e.g., not plugged in), k3s falls back to the pen drive's default data dir — the node
still works, just slower.

---

## Disk detection logic

The SSD is detected as the **largest non-boot disk**. On a Pi booting from a USB pen drive,
the pen drive is `sda` and the SSD is `sdb` (or `sdc` if multiple USB devices are connected).

The detection logic in `05_ssd_storage.yaml`:
1. Find the disk containing the `COS_STATE` or `COS_PERSISTENT` partition → this is the
   boot disk.
2. Among remaining disks, pick the one with the largest byte count.
3. Wipe it, partition it, format with `ext4 -L kairos-ssd`.

Using a partition **label** (`LABEL=kairos-ssd`) instead of UUID means the same
cloud-config works on every Pi in the fleet — each Pi's SSD gets the same label at
format time.

---

## Operational notes

### Replacing the SSD

If you swap the SSD with a new one:
1. The new SSD has no `kairos-ssd` label → `05_ssd_storage.yaml` auto-detects, formats,
   and mounts it on next boot.
2. k3s starts fresh on the new SSD — container images are re-pulled, PVC data is gone.
3. Prometheus history resets; Grafana dashboards are all in ConfigMaps so nothing is lost.

### Backing up Prometheus data

PVC data is at `/usr/local/ssd/k3s-storage/`. To snapshot Prometheus:
```bash
sudo tar czf /tmp/prometheus-backup.tar.gz \
  /usr/local/ssd/k3s-storage/monitoring/
```

### What happens if the SSD is not plugged in

- `usr-local-ssd.mount` fails silently (the unit has no `WantedBy` for k3s, only `Before`).
- k3s starts using its default data dir on the pen drive.
- The node works but Grafana UI loads slowly (30s+ per JS file).
- Plug in the SSD and reboot to restore SSD-backed operation.

---

## Troubleshooting

### SSD not detected

```bash
lsblk
# Should show sdb (or similar) as the SSD

# Check if it was formatted
blkid -L kairos-ssd

# Check mount unit status
sudo systemctl status usr-local-ssd.mount
sudo journalctl -u usr-local-ssd.mount --no-pager | tail -20
```

### k3s fails to start (data-dir on SSD but SSD not mounted)

```bash
# Verify SSD mount
mountpoint -q /usr/local/ssd && echo "SSD mounted" || echo "NOT MOUNTED"

# Verify k3s config
sudo cat /etc/rancher/k3s/config.yaml

# Check the mount unit started before k3s
sudo systemctl list-dependencies k3s.service | grep ssd
```

If the SSD isn't mounted but k3s config says `data-dir: /usr/local/ssd/k3s-data`, k3s
will fail because the path doesn't exist. Fix: mount the SSD manually and start k3s.

### Confirming container images are on the SSD

```bash
# This should show /usr/local/ssd/k3s-data as the data dir
sudo k3s check-config 2>/dev/null | grep -i "data"

# List containerd images (their layers are on the SSD)
sudo k3s crictl images
```

### Grafana JS files still slow

```bash
# Test from the Pi itself (should be < 100ms)
curl -s --compressed -o /dev/null -w "%{time_total}s\n" \
  http://localhost:3000/public/build/app.*.js

# If slow, check where containerd's snapshots are stored
sudo ls /usr/local/ssd/k3s-data/agent/containerd/io.containerd.snapshotter.v1.overlayfs/
```
