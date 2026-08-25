# Phase 3: Real-world demo — monitoring, fleet, and automated upgrades

Evolving the Pi from a bare immutable OS into a real-world Kairos case study: a monitoring stack deployed as k8s workloads via Kairos-native cloud-config stages, a second Pi joining as a k3s agent, and automated upgrades from GitHub Actions.

## Goal

Turn the single-node Kairos Pi into a working demo that shows how Kairos is used in real-world edge deployments:

- **Workload:** A Prometheus + Grafana monitoring stack running on k3s.
- **Observability:** The monitoring stack monitors the Pi(s) themselves — CPU, memory, disk, k3s health.
- **Fleet:** A second Pi joins as a k3s agent, with the monitoring stack automatically extending to it.
- **Automation:** Upgrades triggered from GitHub Actions — no manual SSH needed.

## Architecture

```
   GitHub repo (shashankpai/kairos-test)
        |
        |--- image/Dockerfile ----------------------+
        |--- k8s/*.yaml (monitoring stack)          |  GitHub Actions
        |--- cloud-config/*.yaml                    |  builds arm64 image
        |                                            |  -> GHCR
        v                                            v
   GHCR: ghcr.io/shashankpai/kairos-pi:<tag>
        |
        | (automated: GitHub Actions SSH upgrade on tag push)
        v
   Pi 1 (server, 192.168.1.34) <---> Pi 2 (agent, 192.168.1.35)
        |                              |
        |  on every boot:              |  on every boot:
        |  1. curl+tar pulls repo      |  1. curl+tar pulls repo
        |  2. network stage applies    |  2. network stage applies
        |     k8s manifests to k3s     |     k8s manifests to k3s
        |  3. reconcile stage (60m)    |  3. reconcile stage (60m)
        |     re-applies for drift     |     re-applies for drift
        v                              v
   k3s cluster (2 nodes)
        |
        |--- monitoring namespace
        |    |--- prometheus (scrapes metrics)
        |    |--- grafana (dashboards, NodePort 30300)
        |    |--- node-exporter (DaemonSet, one per node)
        |    |--- kube-state-metrics (k8s object state)
```

---

## Sub-phase 3A: Monitoring stack (workload + observability)

### Design decision: Kairos-native deployment, not baked into the image

The monitoring stack is **NOT** baked into the OS image. The Dockerfile stays minimal — just hadron + a version stamp. Instead, the monitoring components run as **k8s workloads** (containers/pods) on k3s, pulled from public registries (docker.io, quay.io, registry.k8s.io) at runtime via containerd.

This separation is intentional and follows Kairos best practices:

| Layer | What | How |
| --- | --- | --- |
| OS image | hadron + k3s + version stamp | `image/Dockerfile` → GitHub Actions → GHCR |
| Node config | users, SSH keys, network, services | `cloud-config/*.yaml` → boot-time curl+tar pull |
| Workloads | Prometheus, Grafana, etc. | `k8s/*.yaml` → Kairos stages apply to k3s |

Baking workloads into the image would couple application updates to OS updates — you'd need a new OS image every time you change a dashboard. By keeping workloads as k8s manifests in the repo, you can update them with a simple git push + reboot (or reconcile cycle), no image rebuild needed.

### Design decision: Kairos-native stages, not Flux

Instead of installing Flux or ArgoCD for GitOps-style reconciliation, we use Kairos's built-in cloud-config stages:

- **`network` stage** — runs when the network is available. Waits for k3s to be ready, then applies all k8s manifests from the pulled repo.
- **`reconcile` stage** — runs 5 minutes after boot, then every 60 minutes. Re-applies all manifests, correcting any drift (e.g., if someone manually deleted a pod or changed a config).

This is the Kairos-native GitOps pattern. The cloud-config itself is the reconciliation loop — no extra controller pod consuming RAM, no extra moving parts. The `reconcile` stage is Kairos's built-in equivalent of Flux's continuous reconciliation.

### How the manifests reach the cluster

The flow, step by step:

1. **Boot** — Pi boots from the active A/B partition.
2. **`boot` stage** — The `10_git-pull.yaml` cloud-config runs `curl+tar` to download this repo to `/oem/cloud-config-files/`. This includes both `cloud-config/*.yaml` and `k8s/*.yaml`. It then **syncs** the repo's `cloud-config/*.yaml` files into `/oem/` itself, so any new or changed cloud-config files are picked up on the next boot. This makes the cloud-config **self-updating** — you never need to SSH in to add a new cloud-config file. Just push to the repo, reboot the Pi, and it converges.
3. **`network` stage** — The `20_k8s_workloads.yaml` cloud-config waits for k3s to be ready (polls `kubectl get nodes` for up to 5 minutes), then runs `kubectl apply -f /oem/cloud-config-files/k8s/` to deploy all workloads.
4. **`reconcile` stage** — 5 minutes after boot, then every 60 minutes, the same `kubectl apply` runs again. If a pod was deleted, a config was changed, or a new manifest was added to the repo (and pulled on the next boot), the cluster converges back to the desired state.

### Self-updating cloud-config (production pattern)

In a production Kairos deployment, the repo is the single source of truth for **everything** — not just k8s workloads but also the cloud-config files themselves. The `10_git-pull.yaml` stage achieves this by:

1. Downloading the repo (including `cloud-config/*.yaml`)
2. Copying `cloud-config/*.yaml` from the downloaded repo into `/oem/`

This means:
- **Add a new cloud-config file** → push it to `cloud-config/` in the repo → reboot the Pi → it's automatically picked up
- **Change an existing cloud-config file** → push the change → reboot → the updated version is applied
- **No manual SSH** needed to update cloud-config (except for the initial bootstrap at flash time)

The only file that must be written manually (at flash time) is `10_git-pull.yaml` itself — it's the bootstrap that makes everything else self-updating. This is the chicken-and-egg resolution: the flash-time config contains `10_git-pull.yaml`, which then pulls the repo (including an updated `10_git-pull.yaml`) and syncs it to `/oem/`.

**Safety:** The sync only copies files from the repo's `cloud-config/` directory. It does NOT touch `90_custom.yaml` (the flash-time config with secrets like the initial password and k3s token) or any other file in `/oem/` that isn't in the repo.

### Files created

#### `k8s/namespace.yaml`
Creates the `monitoring` namespace with privileged pod security enforcement (needed for node-exporter's hostPath mounts).

#### `k8s/prometheus.yaml`
- **ServiceAccount + ClusterRole + ClusterRoleBinding** — Prometheus needs to discover scrape targets (nodes, pods, services) via the k8s API.
- **ConfigMap** — Prometheus configuration with scrape jobs for:
  - `prometheus` (self-monitoring)
  - `kubernetes-nodes` (kubelet metrics on each node)
  - `node-exporter` (hardware metrics, discovered via pod labels)
  - `kube-state-metrics` (k8s object state)
  - `kubernetes-pods` (any pod with `prometheus.io/scrape: "true"` annotation)
- **PVC** — 2Gi persistent volume for time-series data (7-day retention, using k3s's built-in local-path provisioner).
- **Deployment** — 1 replica, resource limits (512Mi memory), mounts the ConfigMap and PVC.
- **Service** — ClusterIP on port 9090.

#### `k8s/node-exporter.yaml`
- **DaemonSet** — runs one pod per node, exposing hardware metrics (CPU, memory, disk, network) on port 9100.
- Uses `hostPID`, `hostNetwork`, and `hostPath` mounts for `/proc`, `/sys`, and `/` to access the host's hardware state.
- Annotated with `prometheus.io/scrape: "true"` so Prometheus discovers it automatically.
- When the second Pi joins (Phase 3B), node-exporter will automatically schedule on it — no manual intervention.

#### `k8s/kube-state-metrics.yaml`
- **ServiceAccount + ClusterRole + ClusterRoleBinding** — read-only access to all k8s objects (pods, deployments, nodes, etc.).
- **Deployment** — 1 replica, exposes metrics on port 8080.
- **Service** — ClusterIP on port 8080.

#### `k8s/grafana.yaml`
- **PVC** — 1Gi persistent volume for Grafana's database and dashboard state.
- **ConfigMap (datasources)** — auto-provisions Prometheus as the default datasource. No manual Grafana setup needed.
- **ConfigMap (dashboards-provider)** — configures Grafana to load dashboards from `/var/lib/grafana/dashboards/`.
- **ConfigMap (dashboard)** — a custom "Node Exporter Full" dashboard with panels for CPU usage, memory available, disk space, and network traffic. Auto-provisioned on startup.
- **Deployment** — 1 replica, admin/admin credentials (change for production), liveness and readiness probes.
- **Service** — NodePort 30300, accessible at `http://<pi-ip>:30300`.

#### `cloud-config/20_k8s_workloads.yaml`
The Kairos-native stage that ties it all together:

```yaml
stages:
  network:
    - name: "Apply k8s workloads"
      commands:
        - |
          set -e
          # Wait for k3s API to be ready (up to 5 minutes)
          for i in $(seq 1 60); do
            if sudo k3s kubectl get nodes >/dev/null 2>&1; then break; fi
            sleep 5
          done
          # Apply all manifests from the pulled repo
          if [ -d /oem/cloud-config-files/k8s ]; then
            sudo k3s kubectl apply -f /oem/cloud-config-files/k8s/
          fi
  reconcile:
    - name: "Reconcile k8s workloads"
      commands:
        - |
          set -e
          if [ -d /oem/cloud-config-files/k8s ]; then
            sudo k3s kubectl apply -f /oem/cloud-config-files/k8s/
          fi
```

### Resource considerations

| Component | CPU request | Memory request | Memory limit |
| --- | --- | --- | --- |
| Prometheus | 100m | 256Mi | 512Mi |
| Grafana | 50m | 128Mi | 256Mi |
| node-exporter | 50m | 32Mi | 64Mi |
| kube-state-metrics | 50m | 64Mi | 128Mi |
| **Total** | **250m** | **480Mi** | **960Mi** |

On top of k3s's ~500MB baseline, this adds ~500MB. A Pi 4 with 4GB+ RAM handles this comfortably. If the Pi has only 2GB, it will still work but with less headroom — monitor memory usage in Grafana itself.

### Steps performed

1. Created `k8s/` directory with 5 manifest files (namespace, prometheus, node-exporter, kube-state-metrics, grafana).
2. Created `cloud-config/20_k8s_workloads.yaml` with `network` and `reconcile` stages.
3. Committed and pushed to GitHub.
4. On the Pi: updated `/oem/` with the new cloud-config file, ran `sudo kairos-agent run-stage network` to apply manifests immediately.
5. Verified all pods Running: `sudo k3s kubectl get pods -n monitoring`.
6. Accessed Grafana at `http://192.168.1.34:30300` — admin/admin.
7. Verified the auto-provisioned Node Exporter Full dashboard shows Pi metrics.

### Verification commands

```bash
# All monitoring pods should be Running
sudo k3s kubectl get pods -n monitoring

# Prometheus should be scraping targets
sudo k3s kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Then open http://localhost:9090/targets — all targets should be UP

# Grafana is accessible on the Pi's IP
curl -s http://192.168.1.34:30300/api/health
# Expected: {"database":"ok","version":"11.2.0"}

# Test the reconcile stage manually
sudo kairos-agent run-stage reconcile

# Check that manifests were applied from the pulled repo
sudo ls /oem/cloud-config-files/k8s/
# Should show: grafana.yaml  kube-state-metrics.yaml  namespace.yaml  node-exporter.yaml  prometheus.yaml
```

### Troubleshooting

#### Pods stuck in Pending

**Cause:** PVC not being fulfilled. k3s's local-path provisioner needs the `local-path` StorageClass to be present.

**Fix:**
```bash
sudo k3s kubectl get storageclass
# Should show: local-path (default)
```

If not present, k3s should have created it automatically. If not:
```bash
sudo k3s kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml
```

#### node-exporter pod in CrashLoopBackOff

**Cause:** The privileged pod security admission policy is blocking the DaemonSet.

**Fix:** The namespace has `pod-security.kubernetes.io/enforce: privileged` — verify it's set:
```bash
sudo k3s kubectl get namespace monitoring --show-labels
```

#### Prometheus not scraping node-exporter

**Cause:** The `kubernetes-pods` scrape job requires the `prometheus.io/scrape: "true"` annotation on the pod. node-exporter has this, but the service discovery might take a moment.

**Fix:**
```bash
# Check that node-exporter pods have the annotation
sudo k3s kubectl get pods -n monitoring -l app=node-exporter -o jsonpath='{.items[0].metadata.annotations}'
# Should show: {"prometheus.io/port":"9100","prometheus.io/scrape":"true"}

# Check Prometheus targets
sudo k3s kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open http://localhost:9090/targets — node-exporter should be UP
```

#### Grafana dashboard not auto-provisioned

**Cause:** The dashboard ConfigMap might not be loaded if Grafana started before the ConfigMap was applied.

**Fix:** Restart Grafana after the manifests are applied:
```bash
sudo k3s kubectl rollout restart deployment grafana -n monitoring
```

#### `network` stage fails with "k3s kubectl: command not found"

**Cause:** The `network` stage runs before k3s is fully initialized, or the `k3s` binary isn't in PATH for the stage's shell.

**Fix:** The stage waits up to 5 minutes for k3s to be ready. If k3s takes longer (slow pen drive), the stage will exit but the `reconcile` stage will apply the manifests on the next cycle (5 min after boot). You can also trigger it manually:
```bash
sudo kairos-agent run-stage network
```

---

## Sub-phase 3B: Second Pi as k3s agent (fleet)

### Goal

Add a second Raspberry Pi 4 as a k3s agent, joining the first Pi's cluster. This demonstrates the real Kairos fleet story: shared cloud-config, shared image, k3s token-based join, and coordinated monitoring across nodes.

### Steps

1. **Get the k3s node-token from Pi 1:**
   ```bash
   ssh kairos@192.168.1.34
   sudo cat /var/lib/rancher/k3s/server/node-token
   ```

2. **Flash Pi 2** with the v0.1.0 `.raw.gz` from the GitHub Release (same as Phase 1, but using our custom image instead of stock hadron).

3. **Pi 2's flash-time cloud-config** (in `build/cloud-config.yaml`, gitignored):
   ```yaml
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
     agent: true
     server: https://192.168.1.34:6443
     token: <node-token from Pi 1>
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

4. **Set static IP on Pi 2** (`192.168.1.35`) — same as Phase 2's k3s IP fix, to avoid the IP-change issue:
   ```bash
   sudo bash -c 'cat > /etc/systemd/network/10-end0.network << "ENDOFFILE"
   [Match]
   Name=end0

   [Network]
   DHCP=no
   Address=192.168.1.35/24
   Gateway=192.168.1.1
   DNS=192.168.1.1
   DNS=8.8.8.8
   ENDOFFILE'
   sudo systemctl restart systemd-networkd
   ```

5. **Boot Pi 2** — it will join the k3s cluster automatically on first boot.

6. **Verify on Pi 1:**
   ```bash
   sudo k3s kubectl get nodes -o wide
   # Should show 2 nodes: kairos-XXXX (server) and kairos-YYYY (agent)

   # node-exporter DaemonSet should schedule on both nodes automatically
   sudo k3s kubectl get pods -n monitoring -l app=node-exporter -o wide
   # Should show one pod per node

   # Grafana should now show both nodes in the Node Exporter Full dashboard
   ```

### Security note

The k3s node-token is a secret that grants cluster access. It must **never** be committed to the repo. It goes in the flash-time `build/cloud-config.yaml` only (gitignored). The repo's `cloud-config/00_base.yaml` stays as the server config; a `cloud-config/01_agent.yaml` template can be added with a placeholder token for documentation purposes.

### Troubleshooting

#### Pi 2 doesn't join the cluster

**Diagnosis:**
```bash
# On Pi 2:
sudo systemctl status k3s-agent 2>/dev/null || sudo systemctl status k3s
sudo journalctl -u k3s-agent --no-pager | tail -20  # or k3s if agent service name differs
```

**Common causes:**
- Wrong server URL or token — verify against Pi 1's `node-token`.
- Firewall blocking port 6443 on Pi 1 — k3s doesn't set up a firewall by default, but check if you added one.
- Pi 1's k3s not running — verify on Pi 1: `sudo k3s kubectl get nodes`.

#### node-exporter not scheduling on Pi 2

**Cause:** The DaemonSet might not have been applied yet, or the node might not be Ready.

**Fix:**
```bash
# Check node status
sudo k3s kubectl get nodes

# Check DaemonSet
sudo k3s kubectl get daemonset -n monitoring node-exporter
sudo k3s kubectl describe daemonset -n monitoring node-exporter

# Force reconcile
sudo kairos-agent run-stage reconcile
```

---

## Sub-phase 3C: Automated upgrades from GitHub Actions

### Goal

Replace manual SSH upgrades with a GitHub Actions workflow that automatically upgrades the Pi(s) when a new `v*` tag is pushed. The flow becomes: push tag → Actions builds image → Actions SSHes into Pi(s) → upgrade → reboot → done.

### Prerequisites

1. **Generate a dedicated SSH key** for GitHub Actions (not your personal key):
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/kairos-actions-key -C "github-actions@kairos-test"
   ```

2. **Add the public key to each Pi** — create a new cloud-config file `cloud-config/30_actions_key.yaml`:
   ```yaml
   #cloud-config
   users:
     - name: kairos
       ssh_authorized_keys:
         - "ssh-ed25519 AAAA... kairos-actions-key"
   ```
   This gets pulled by the boot-time curl+tar and applied on the next boot/reconcile.

3. **Add GitHub secrets:**
   - `KAIROS_SSH_KEY` — the private key contents
   - `KAIROS_PI1_IP` — `192.168.1.34`
   - `KAIROS_PI2_IP` — `192.168.1.35` (when Pi 2 is ready)

### Workflow

A new workflow `.github/workflows/upgrade-pi.yaml` that triggers after a successful tag build:

```yaml
name: Upgrade Pi
on:
  workflow_run:
    workflows: ["Build and publish Kairos Pi image"]
    types: [completed]
    branches: ["v*"]

jobs:
  upgrade:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.KAIROS_SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.KAIROS_PI1_IP }} >> ~/.ssh/known_hosts
      - name: Upgrade Pi 1 (server)
        run: |
          ssh -i ~/.ssh/id_ed25519 kairos@${{ secrets.KAIROS_PI1_IP }} \
            "sudo kairos-agent upgrade --source oci:ghcr.io/shashankpai/kairos-pi:${{ github.event.workflow_run.head_branch }}"
      - name: Wait for reboot and verify
        run: |
          sleep 120
          ssh -i ~/.ssh/id_ed25519 kairos@${{ secrets.KAIROS_PI1_IP }} \
            "grep KAIROS_TEST_VERSION /etc/os-release && sudo k3s kubectl get nodes"
```

### Upgrade order

For a multi-node cluster, the upgrade order matters:
1. **Server first** — upgrade Pi 1, wait for reboot, verify k3s is healthy.
2. **Agents second** — upgrade Pi 2, wait for reboot, verify it rejoins the cluster.

The workflow should reflect this ordering with sequential steps and verification between them.

### Troubleshooting

#### SSH connection from GitHub Actions fails

**Cause:** The Actions SSH key isn't authorized on the Pi, or the Pi's IP changed.

**Fix:**
- Verify the public key is in the Pi's `/oem/` cloud-config: `sudo cat /oem/30_actions.yaml` (or wherever the key was added).
- Verify the Pi's IP is static and matches the GitHub secret.
- Test locally: `ssh -i ~/.ssh/kairos-actions-key kairos@192.168.1.34`

#### Upgrade succeeds but Pi doesn't reboot

**Cause:** `kairos-agent upgrade` sets the passive slot but doesn't trigger a reboot automatically.

**Fix:** Add a reboot command to the SSH command:
```bash
ssh kairos@<pi> "sudo kairos-agent upgrade --source oci:... && sudo reboot"
```

---

## Useful diagnostic commands (Phase 3)

```bash
# Monitoring stack status
sudo k3s kubectl get pods -n monitoring
sudo k3s kubectl get svc -n monitoring
sudo k3s kubectl get pvc -n monitoring

# Prometheus targets
sudo k3s kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open http://localhost:9090/targets

# Grafana health
curl -s http://192.168.1.34:30300/api/health

# Cluster nodes
sudo k3s kubectl get nodes -o wide

# Kairos stages
sudo kairos-agent run-stage network    # apply k8s manifests now
sudo kairos-agent run-stage reconcile  # re-apply for drift correction
sudo journalctl -b | grep -i 'k8s workloads\|Apply k8s'

# Verify repo contents were pulled
sudo ls /oem/cloud-config-files/k8s/
sudo ls /oem/cloud-config-files/cloud-config/
```
