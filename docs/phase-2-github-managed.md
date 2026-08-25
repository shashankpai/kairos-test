# Phase 2: GitHub-managed image builds, config pulls, and upgrades

Expanding the Pi from a standalone flashed node to a GitHub-managed immutable OS: custom image built by GitHub Actions, boot-time config pull from the repo, and A/B upgrades from GHCR.

## Goal

- Build a custom Kairos image from `image/Dockerfile` via GitHub Actions and push it to GHCR.
- Have the Pi pull this repo's cloud-config files on every boot so config changes land after a reboot.
- Upgrade the Pi to the custom GHCR image using Kairos's A/B partition scheme.
- Tag releases to produce flashable `.raw.gz` artifacts for new nodes.

## Prerequisites

- Phase 1 complete: Pi booting stock hadron, k3s running, SSH access working.
- The GitHub repo (`shashankpai/kairos-test`) created and public (public simplifies boot-time pulls — no auth needed).

## Step 1: Repo structure

The repo is organized as:

```
kairos-test/
  image/
    Dockerfile                      # Custom image: FROM hadron + version stamp
  cloud-config/
    00_base.yaml                    # Users, SSH keys, k3s enable
    10_git-pull.yaml                # Boot-time repo download (curl+tar)
  .github/workflows/
    build-and-publish.yaml          # Builds arm64 image, pushes to GHCR, releases on tags
  docs/
    phase-1-flashing.md             # How we flashed the Pi
    phase-2-github-managed.md       # This file
    00-overview.md                  # Architecture
    01-flashing.md                  # Reference flashing doc
    02-upgrades.md                  # Upgrade runbook
    03-config-management.md         # Config management details
    04-troubleshooting.md           # Quick-reference troubleshooting
```

## Step 2: Custom image Dockerfile

The Dockerfile extends the stock hadron image with a custom version stamp so we can identify our image on the Pi:

```dockerfile
FROM quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1

ARG KAIROS_TEST_VERSION=0.1.0
RUN echo "KAIROS_TEST_VERSION=${KAIROS_TEST_VERSION}" >> /etc/os-release
```

> **Why not `kairos-init`?** When extending an existing hadron image, `kairos-init` is not needed — it's only for raw base distros (ubuntu, alpine, etc.). The hadron image is already a valid Kairos artifact.

> **Hadron doesn't ship `apk`:** You cannot install packages at runtime on hadron. The Dockerfile can only stamp files and copy configs. To add packages, you'd need to build from a base distro with kairos-init instead.

## Step 3: GitHub Actions workflow

The workflow (`.github/workflows/build-and-publish.yaml`):

- **On push to `main`:** builds `linux/arm64` image, pushes `:latest` and `:sha-<short>` to GHCR.
- **On `v*` tag push:** builds the image, pushes `:vX.Y.Z` and `:latest`, runs AuroraBoot to produce a `.raw.gz` flash artifact, and creates a GitHub Release with the artifact attached.
- Uses QEMU for arm64 emulation on the amd64 GitHub-hosted runner (slow but works on free runners — ~30 seconds for our small image, longer if the base image needs pulling).

Key workflow details:
- `permissions: contents: write` (for GitHub Release), `packages: write` (for GHCR).
- Uses `GITHUB_TOKEN` (auto-provided) for GHCR auth — no extra secrets needed.
- The `paths` filter was initially set to only trigger on `image/**` changes, but this blocked tag pushes when the tagged commit didn't touch `image/**`. **Fix:** removed the `paths` filter entirely so all `main` pushes and all `v*` tags trigger the build.

## Step 4: Boot-time config pull

The Pi needs to download this repo's cloud-config files on every boot so that config changes pushed to GitHub land on the Pi after a reboot.

### The nogit problem

The initial approach used the Kairos `stages.boot.git` module:

```yaml
# DOES NOT WORK on hadron v0.5.1
stages:
  boot:
    - name: "Pull config repo"
      git:
        url: "https://github.com/shashankpai/kairos-test.git"
        path: "/oem/cloud-config-files"
        branch: "main"
```

This failed with:
```
Error on file  on stage Pull config repo: git plugin not available in nogit build
```

Hadron v0.5.1 is a **"nogit" build** — the kairos-agent's git stage plugin is not compiled in, and `git` is not installed on the system. There's no `apk` either, so you can't install git at runtime.

### The curl+tar solution

Since `curl` IS available on hadron, we download a GitHub tarball instead:

```yaml
#cloud-config
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

This achieves the same result as `git clone` — the repo contents end up in `/oem/cloud-config-files/` — without requiring git.

### Onboarding an existing Pi (no re-flash needed)

The Pi was flashed before this repo existed, so `/oem/` didn't have the git-pull config. Since the repo is public, we can write the file directly over SSH:

```bash
ssh -i ~/.ssh/id_ed25519 kairos@<pi-ip>

sudo bash -c 'cat > /oem/10_git-pull.yaml << "ENDOFFILE"
#cloud-config
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
ENDOFFILE'
```

Test it immediately without rebooting:

```bash
sudo kairos-agent run-stage boot
```

Verify:

```bash
sudo ls /oem/cloud-config-files/cloud-config/
# Should show: 00_base.yaml  10_git-pull.yaml
```

### The chicken-and-egg

`10_git-pull.yaml` defines the repo download, but the download is what delivers `10_git-pull.yaml`. Resolution: the **flash-time** cloud-config must already contain the curl+tar entry. After the first boot, the repo's copy is re-downloaded each boot and stays in sync. If you ever change the download logic, you must also update the flash-time config and re-flash.

## Step 5: Upgrade the Pi to the custom GHCR image

Once the GitHub Actions build is green and the image is on GHCR:

```bash
ssh -i ~/.ssh/id_ed25519 kairos@<pi-ip>

# Run the upgrade — pulls the OCI image and writes it to the passive slot
sudo kairos-agent upgrade --source oci:ghcr.io/shashankpai/kairos-pi:sha-30630bc

# Reboot into the new image
sudo reboot
```

After reboot (~2-3 min), SSH back in and validate:

```bash
# Confirm the custom image is active (this field doesn't exist on stock hadron)
grep KAIROS_TEST_VERSION /etc/os-release
# Expected: KAIROS_TEST_VERSION=sha-30630bc3cf2fd1334afd54358c01414322cbf2e9

# k3s survived the upgrade
sudo k3s kubectl get nodes -o wide

# Boot-time config pull still works
sudo ls /oem/cloud-config-files/cloud-config/
```

Kairos uses A/B partitions: the new image goes to the passive slot and becomes active on reboot. The old image becomes the new passive (fallback). If the new image fails to boot, Kairos automatically rolls back after the boot assessment timeout.

## Step 6: Tag a release

Once the upgrade is verified:

```bash
git tag v0.1.0
git push origin v0.1.0
```

This triggers the release build, which:
1. Pushes `ghcr.io/shashankpai/kairos-pi:v0.1.0` and `:latest` to GHCR.
2. Runs AuroraBoot to produce a `.raw.gz` flash artifact.
3. Creates a GitHub Release v0.1.0 with the `.raw.gz` attached.

The `.raw.gz` can be used to flash fresh Pis with the same image used for upgrades — no need to build locally.

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

## Troubleshooting encountered during Phase 2

### Docker tag format: `+` is invalid

**Problem:** `kairos-agent upgrade list-releases` displays the image as `k3sv1.36.3+k3s1`. Using this in a Docker `FROM` or `docker pull` fails with "invalid reference format".

**Root cause:** The `+` character is invalid in Docker tags. The actual quay.io tag uses hyphens: `k3s-v1.36.3-k3s1` (with a hyphen between `k3s` and `v1.36.3`, and a hyphen instead of `+`).

**Fix:** Always verify the tag against the quay.io API or web UI. The display format from `kairos-agent` is not the pullable tag format.

### `git plugin not available in nogit build`

**Problem:** The `stages.boot.git` cloud-config module fails with:
```
Error on file  on stage Pull config repo: git plugin not available in nogit build
```

**Root cause:** Hadron v0.5.1 is a "nogit" build. The kairos-agent's git stage plugin is not compiled in, and `git` is not installed. There's no `apk` to install it at runtime.

**Diagnosis:**
```bash
# Check the boot stage logs
sudo journalctl -b -u cos-setup-boot.service --no-pager 2>&1 | grep -i 'git\|error\|pull'

# Verify git is not installed
which git  # returns nothing

# Verify the cloud-config is loaded
sudo kairos-agent config  # shows merged config including stages.boot.git
```

**Fix:** Use `stages.boot.commands` with `curl` + `tar` to download a GitHub tarball instead. See the curl+tar solution in Step 4 above.

### k3s stuck in "activating" after IP change

**Problem:** After a reboot, the Pi got a new DHCP IP (192.168.1.34 instead of 192.168.1.26). k3s had the old IP baked into its certificates and config, so it couldn't connect to itself:

```
dial tcp 192.168.1.26:6443: connect: no route to host
```

k3s was stuck in a restart loop, `systemctl is-active k3s` showing `activating` indefinitely.

**Diagnosis:**
```bash
# Check the Pi's current IP
ip addr show end0 | grep 'inet '

# Check k3s errors
sudo journalctl -b -u k3s --no-pager 2>&1 | grep -iE 'error|fatal|fail' | tail -10
# Look for "no route to host" or "connection refused" to the old IP
```

**Fix:** Set a static IP and reset k3s state:

```bash
# 1. Set static IP via systemd-networkd
sudo bash -c 'cat > /etc/systemd/network/10-end0.network << "ENDOFFILE"
[Match]
Name=end0

[Network]
DHCP=no
Address=192.168.1.34/24
Gateway=192.168.1.1
DNS=192.168.1.1
DNS=8.8.8.8
ENDOFFILE'

# 2. Stop k3s
sudo systemctl stop k3s

# 3. Delete k3s state (safe for single-node — workloads reschedule automatically)
sudo rm -rf /var/lib/rancher/k3s/server
sudo rm -rf /var/lib/rancher/k3s/agent

# 4. Restart networking with the static IP
sudo systemctl restart systemd-networkd

# 5. Verify the static IP
ip addr show end0 | grep 'inet '

# 6. Start k3s fresh
sudo systemctl start k3s

# 7. Wait and verify
sleep 30
sudo systemctl is-active k3s
sudo k3s kubectl get nodes -o wide
```

**Prevention:** Always set a static IP on an edge node before relying on k3s. DHCP IP changes break k3s because the server certificate is bound to the IP at initialization time.

### GitHub Actions workflow not triggering on tags

**Problem:** Pushing a `v*` tag didn't trigger the release build.

**Root cause:** The workflow had a `paths` filter that only triggered on changes to `image/**` and the workflow file itself. When the tagged commit only changed docs, the filter blocked the trigger.

**Fix:** Removed the `paths` filter so all `main` pushes and all `v*` tags trigger the build:

```yaml
# Before (broken):
on:
  push:
    branches: [main]
    paths:
      - 'image/**'
      - '.github/workflows/build-and-publish.yaml'
    tags: ['v*']

# After (fixed):
on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:
```

### Boot-time config pull not creating files

**Problem:** After writing `/oem/10_git-pull.yaml` and rebooting, `/oem/cloud-config-files/` was empty.

**Diagnosis steps:**
1. Verify the file was written correctly: `sudo cat /oem/10_git-pull.yaml` — ensure the `#cloud-config` header is present.
2. Check the merged config: `sudo kairos-agent config` — ensure the `stages.boot` entry appears.
3. Check the boot stage service logs: `sudo journalctl -b -u cos-setup-boot.service --no-pager`
4. Look for the specific stage: `sudo journalctl -b -u cos-setup-boot.service --no-pager | grep -i 'Pull config'`

In our case, the log showed the `git plugin not available in nogit build` error (see above). The fix was switching to curl+tar.

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

### Useful diagnostic commands

```bash
# Full kairos-agent log from this boot
sudo journalctl -b -u kairos-agent --no-pager

# Boot stage service (processes stages.boot on every boot)
sudo journalctl -b -u cos-setup-boot.service --no-pager

# Immucore log (initramfs stages: rootfs, initramfs)
sudo journalctl -b -t immucore --no-pager

# Merged cloud-config the system loaded
sudo kairos-agent config

# Test a stage without rebooting
sudo kairos-agent run-stage boot

# k3s status and logs
sudo systemctl status k3s
sudo journalctl -b -u k3s --no-pager | tail -30
sudo k3s kubectl get nodes -o wide

# Network configuration
ip addr show end0
sudo cat /etc/systemd/network/10-end0.network
```

## Useful files on the Pi

| Path | Contents |
| --- | --- |
| `/oem/90_custom.yaml` | The flash-time cloud-config (written by AuroraBoot). |
| `/oem/10_git-pull.yaml` | The boot-time repo download config (written manually in Phase 2). |
| `/oem/cloud-config-files/` | Repo contents downloaded at boot. |
| `/usr/local/.kairos/deployed` | Sentinel; presence means first-boot setup is done. |
| `/etc/sysconfig/k3s` | k3s config written by kairos-agent. |
| `/etc/systemd/network/10-end0.network` | Static IP config (added in Phase 2). |
| `/run/initramfs/cos-state/cOS/` | Active/passive OS images (recovery only). |
