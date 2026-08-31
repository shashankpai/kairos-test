# Phase 4: OPNsense + Managed Switch Network Migration

Migrating the homelab from a flat 192.168.1.x network to a segmented setup with OPNsense
providing DHCP, firewall, and static IP mappings on 10.0.0.x for homelab devices.

---

## Current Network (Before Migration)

```
Main router (192.168.1.1, ZTE, home DHCP)
  │
  └── WiFi → Range extender (1 RJ45 port, repeater mode)
               │
               └── Mercury 8-port unmanaged switch
                     ├── Proxmox 1 (192.168.1.78)
                     ├── Proxmox 2 (192.168.1.x)
                     ├── Proxmox 3 (192.168.1.x)
                     ├── Proxmox 4 (192.168.1.x)
                     ├── Ubuntu controller (192.168.1.32)
                     └── Pi 1 (192.168.1.x, dynamic — the problem)
```

**Problem:** Pi gets a random DHCP IP on every boot. Kairos `write_files` for static IP
does not work reliably on the immutable rootfs. No router-level DHCP reservation available.
All homelab devices are on the same subnet as home devices — no isolation.

---

## Target Network (After Migration)

```
Main router (192.168.1.1, ZTE — unchanged)
  │
  └── WiFi → Range extender
               │
               └── 5-port managed switch (TP-Link SG105E, 802.1Q VLAN)
                     ├── Port 1: VLAN 1 → extender uplink
                     ├── Port 2: Trunk (VLAN 1+10) → Proxmox 1
                     │     ├── vmbr0 (VLAN 1) → 192.168.1.78 (existing VMs, management)
                     │     ├── vmbr1 (VLAN 10) → 10.0.0.78 (homelab management)
                     │     └── OPNsense VM
                     │           ├── WAN: vmbr0 → 192.168.1.x (from home router)
                     │           └── LAN: vmbr1 → 10.0.0.1 (homelab DHCP)
                     ├── Port 3: VLAN 10 → Pi 1 (10.0.0.34, static via OPNsense)
                     ├── Port 4: VLAN 10 → Ubuntu controller (10.0.0.32, static)
                     └── Port 5: VLAN 1 → Mercury switch
                           ├── Proxmox 2 (192.168.1.x — unchanged)
                           ├── Proxmox 3 (192.168.1.x — unchanged)
                           └── Proxmox 4 (192.168.1.x — unchanged)
```

### What stays unchanged

- Home router (192.168.1.1) — still provides internet, DHCP for home devices
- Range extender — still bridges WiFi to wired
- Proxmox 2, 3, 4 — stay on 192.168.1.x via Mercury switch
- All existing VMs on Proxmox 2, 3, 4 — unchanged
- All existing VMs on Proxmox 1 attached to vmbr0 — unchanged
- Home devices (phones, TV, laptop) — unchanged

### What changes

- Proxmox 1: gets vmbr1 (VLAN 10 bridge), one cable carries both VLANs
- Pi 1: moves to managed switch Port 3, gets 10.0.0.34 from OPNsense
- Ubuntu: moves to managed switch Port 4, gets 10.0.0.32 from OPNsense
- New OPNsense VM on Proxmox 1: routes between VLAN 1 and VLAN 10

---

## Hardware

### What to buy

| Item | Spec | Example | Est. Cost (India) |
|------|------|---------|-------------------|
| Managed switch | 5-port, Gigabit, 802.1Q VLAN | TP-Link TL-SG105E v4+ | ~₹1,500 |

### Verification checklist before buying

- [ ] Gigabit (1000 Mbps) — NOT 10/100
- [ ] 802.1Q VLAN support (tag-based, not just port-based)
- [ ] Web UI management
- [ ] At least 5 ports

### Avoid

- Any switch labeled "Fast Ethernet" or "10/100"
- Any switch that only says "VLAN" without "802.1Q" or "tag-based"
- TP-Link TL-SG105E v1-v3 (port-based VLAN only, no web UI — need v4+)

---

## Step 1: Initial Switch Setup

### 1.1 Access the switch web UI

The TP-Link SG105E has a default IP of `192.168.0.1`. To access it for the first time:

**Option A — Direct connection with static IP (recommended):**

1. Unplug all cables from the switch
2. Connect a laptop/Ubuntu machine to **Port 1** via Ethernet
3. Set the machine to a static IP on the 192.168.0.x subnet:

```bash
# On Ubuntu:
sudo ip addr add 192.168.0.2/24 dev eth0
# (replace eth0 with your actual interface — check with `ip link`)

# On macOS:
sudo ifconfig en0 192.168.0.2 netmask 255.255.255.0
```

4. Open a browser and go to `http://192.168.0.1`
5. Login: `admin` / `admin` (TP-Link default)

**Option B — If 192.168.0.1 doesn't respond (factory reset):**

1. Unplug ALL cables from the switch
2. Power off the switch (unplug power adapter)
3. Find the Reset button (small pinhole on the side/bottom)
4. Use a paperclip — press and hold Reset
5. Plug in power WHILE holding Reset
6. Hold for 10 seconds, then release
7. Plug laptop into Port 1
8. Set laptop to 192.168.0.2
9. Try `http://192.168.0.1` again

### 1.2 Set management IP on home subnet

Once logged in, change the switch's IP to your home subnet so it's reachable after you
connect it to the real network:

```
System → IP Setting → IP Configuration
  ├── IP Assignment: Static
  ├── IP Address: 192.168.1.250
  ├── Subnet Mask: 255.255.255.0
  └── Gateway: 192.168.1.1
```

Save. **The switch is now at 192.168.1.250 — the UI will become unreachable because your
machine is still on 192.168.0.x.**

### 1.3 Restore your machine's IP and reconnect

```bash
# On Ubuntu:
sudo ip addr del 192.168.0.2/24 dev eth0
sudo ip addr add 192.168.1.32/24 dev eth0

# OR simply plug the extender cable into Port 1 of the switch
# — Ubuntu will get a DHCP IP on 192.168.1.x from the home router
```

Now access the switch at `http://192.168.1.250` from any device on the home network.

### Troubleshooting: Can't access the switch UI

| Problem | Cause | Fix |
|---------|-------|-----|
| `192.168.0.1` doesn't respond | Switch was previously configured to a different IP | Factory reset (hold Reset button while powering on) |
| `192.168.1.250` doesn't respond after changing IP | Your machine is still on 192.168.0.x | Change machine IP to 192.168.1.x, or plug extender into Port 1 for DHCP |
| No web UI at all (any IP) | SG105E v1-v3 has no web UI | Check version on sticker — v1-v3 needs Windows-only "Easy Smart Configuration Utility" |
| `192.168.1.1` shows ZTE router, not TP-Link | That's your main router, not the switch | Switch is at `192.168.1.250` (or `192.168.0.1` if not yet configured) |

### How to check the hardware version

Look at the **sticker on the back or bottom** of the switch:

```
Model: TL-SG105E
Version: 4.0    ← v4+ has web UI + 802.1Q VLAN
Ver: 2.0        ← v1-v3: NO web UI, Windows utility only, port-based VLAN only
```

**If you have v1-v3:** Return it and buy v4+ or a Netgear GS305T (all versions have web UI).

---

## Step 2: Configure VLANs

### 2.1 Create VLANs

In the switch web UI at `http://192.168.1.250`:

```
VLAN → 802.1Q VLAN → VLAN Config → Add
```

Create two VLANs:

| VLAN ID | Description | Subnet |
|---------|-------------|--------|
| 1 | home | 192.168.1.0/24 |
| 10 | homelab | 10.0.0.0/24 |

(VLAN 1 usually exists by default — just add VLAN 10.)

### 2.2 Assign ports to VLANs

**A) 802.1Q VLAN → VLAN Config** (which ports are members of which VLAN):

```
VLAN 1:  Port 1 (Untagged), Port 2 (Tagged), Port 5 (Untagged)
VLAN 10: Port 2 (Tagged), Port 3 (Untagged), Port 4 (Untagged)
```

**B) 802.1Q PVID Setting** (sets the PVID for untagged incoming traffic):

```
Port 1: PVID 1
Port 2: PVID 1
Port 3: PVID 10
Port 4: PVID 10
Port 5: PVID 1
```

### 2.3 Verify

After creating VLAN 10 and editing VLAN 1, the VLAN table should look like:

```
VLAN ID  VLAN Name  Member Ports  Tagged Ports  Untagged Ports
1        Default    1,2,5         2             1,5
10       homelab    2-4           2             3-4
```

### 2.4 Set PVID for each port

Navigate to **802.1Q PVID Setting** (separate page from VLAN membership).

The PVID tells the switch which VLAN to assign when untagged traffic enters a port.
By default all ports will show PVID 1 — you need to change Ports 3 and 4 to PVID 10.

Set each port:

| Port | PVID | Why |
|------|------|-----|
| 1 | 1 | Extender traffic → VLAN 1 (leave as default) |
| 2 | 1 | Trunk port default (leave as default — Proxmox handles tagging) |
| 3 | 10 | **Change from 1 to 10** — Pi traffic → VLAN 10 |
| 4 | 10 | **Change from 1 to 10** — Ubuntu traffic → VLAN 10 |
| 5 | 1 | Mercury switch traffic → VLAN 1 (leave as default) |

After applying, the PVID table should look like:

```
Port 1: PVID 1
Port 2: PVID 1
Port 3: PVID 10
Port 4: PVID 10
Port 5: PVID 1
```

### 2.5 Final verification

The switch is fully configured when both tables match:

**VLAN membership:**
```
VLAN 1:  Ports 1(U), 2(T), 5(U)
VLAN 10: Ports 2(T), 3(U), 4(U)
```

**PVID:**
```
Port 1: PVID 1
Port 2: PVID 1
Port 3: PVID 10
Port 4: PVID 10
Port 5: PVID 1
```

### Port assignment summary

| Port | PVID | Untagged | Tagged | Connected to |
|------|------|----------|--------|-------------|
| 1 | 1 | VLAN 1 | — | Extender uplink (home network) |
| 2 | 1 | — | VLAN 1, VLAN 10 | Proxmox 1 (trunk — carries both VLANs) |
| 3 | 10 | VLAN 10 | — | Pi 1 (homelab) |
| 4 | 10 | VLAN 10 | — | Ubuntu controller (homelab) |
| 5 | 1 | VLAN 1 | — | Mercury switch (Proxmox 2,3,4 — home network) |

### How VLAN port assignment works

Devices don't "know" about VLANs — the switch handles it:

- **Untagged ports** (1, 3, 4, 5): The switch silently adds/strips VLAN tags. The device
  just sees a normal network. Port 3 sends all traffic on VLAN 10 — the Pi has no idea
  VLANs exist.
- **Tagged/trunk port** (2): Carries multiple VLANs with tags intact. Proxmox sorts
  traffic by VLAN tag into different virtual bridges (vmbr0 for VLAN 1, vmbr1 for VLAN 10).

### VLAN explanation: what we configured and why

#### The two VLANs

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 1 | Default (home) | 192.168.1.x | Existing home network — internet, phones, TV, existing Proxmox VMs |
| 10 | homelab | 10.0.0.x | New isolated network for Pi, Ubuntu controller, and future homelab VMs |

#### Port-by-port breakdown

**Port 1 — Untagged on VLAN 1**
- Connected to: range extender (uplink to home router / internet)
- Traffic comes in untagged from the extender → switch tags it as VLAN 1 internally
- This is how the homelab reaches the internet: VLAN 10 → OPNsense → VLAN 1 → Port 1 → extender → home router → internet

**Port 2 — Tagged on both VLAN 1 and VLAN 10 (TRUNK)**
- Connected to: Proxmox 1
- This is the special port — it carries BOTH VLANs simultaneously
- Proxmox receives tagged traffic and sorts it:
  - VLAN 1 tags → vmbr0 → 192.168.1.x (home network, existing VMs, management)
  - VLAN 10 tags → vmbr1 → 10.0.0.x (homelab, OPNsense LAN)
- One physical cable, two networks — Proxmox handles the separation internally

**Port 3 — Untagged on VLAN 10 only**
- Connected to: Pi 1
- Pi sends/receives plain Ethernet (no VLAN tags — Pi doesn't know VLANs exist)
- Switch silently tags all Port 3 traffic as VLAN 10
- Pi will get 10.0.0.34 from OPNsense's DHCP server

**Port 4 — Untagged on VLAN 10 only**
- Connected to: Ubuntu controller
- Same as Port 3 — Ubuntu doesn't know about VLANs
- Switch tags all traffic as VLAN 10
- Ubuntu will get 10.0.0.32 from OPNsense's DHCP server

**Port 5 — Untagged on VLAN 1 only**
- Connected to: Mercury unmanaged switch (which has Proxmox 2, 3, 4)
- All traffic on this port is VLAN 1 (home network)
- Proxmox 2, 3, 4 stay on 192.168.1.x — completely unchanged

#### The key concept: Tagged vs Untagged vs Not Member

| Term | Meaning | Where used |
|------|---------|-----------|
| **Untagged** | Switch adds/strips VLAN tag silently. Device sees normal network. | End devices (Pi, Ubuntu, extender, Mercury) |
| **Tagged** | Switch keeps VLAN tags in the packet. Device must understand VLANs. | Trunk ports (Proxmox — it sorts by tag) |
| **Not Member** | Port doesn't participate in this VLAN at all. | Ports 3,4 not on VLAN 1; Ports 1,5 not on VLAN 10 |

#### Traffic flow example: Pi requesting an IP

```
1. Pi powers on → sends DHCP request (plain Ethernet, no VLAN tag)
2. Switch Port 3 receives it → tags it as VLAN 10 internally
3. Switch forwards to Port 2 (the only other VLAN 10 member) with VLAN 10 tag
4. Proxmox NIC receives it → vmbr1 (VLAN 10) → OPNsense LAN interface sees it
5. OPNsense responds: "Your IP is 10.0.0.34" (sends with VLAN 10 tag)
6. Proxmox → Port 2 → switch forwards to Port 3 with VLAN 10 tag
7. Switch Port 3 strips the tag → sends to Pi as plain Ethernet
8. Pi receives: "Your IP is 10.0.0.34" — Pi has no idea VLANs were involved
```

#### Traffic flow example: Pi reaching the internet

```
1. Pi (10.0.0.34) sends packet to 8.8.8.8
2. Switch tags as VLAN 10 → forwards to Port 2 → Proxmox → vmbr1 → OPNsense LAN
3. OPNsense routes: VLAN 10 (LAN) → VLAN 1 (WAN)
4. OPNsense sends out on vmbr0 → Proxmox → Port 2 (tagged VLAN 1)
5. Switch forwards VLAN 1 to Port 1 (extender) → strips tag
6. Extender → home router → internet
7. Response comes back the same way in reverse
```

Some TP-Link switches auto-set PVID based on the untagged VLAN membership, but the
SG105E requires manual PVID configuration. Verify in the **802.1Q PVID Setting** page
that Ports 3 and 4 show PVID 10 (see Step 2.4 above).

---

## Step 3: Create vmbr1 on Proxmox 1

SSH to Proxmox 1 (still at 192.168.1.78, still on Mercury switch — no cable changes yet):

```bash
ssh root@192.168.1.78
```

Backup and edit `/etc/network/interfaces`:

```bash
cp /etc/network/interfaces /etc/network/interfaces.backup

cat >> /etc/network/interfaces << 'EOF'

# Homelab VLAN 10 bridge
auto vmbr1
iface vmbr1 inet static
    address 10.0.0.78/24
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10
EOF
```

**Do NOT apply yet.** We'll apply after the cable swap.

> **Note:** Replace `enp3s0` with your actual physical NIC name. Check with `ip link` on
> Proxmox 1.

---

## Step 4: Download OPNsense ISO

```bash
# On Proxmox 1, download OPNsense DVD ISO
cd /var/lib/vz/template/iso
wget https://mirror.opnsense.org/releases/24.7/OPNsense-24.7-dvd-amd64.iso
```

(Use the latest stable version from https://opnsense.org/download/)

---

## Step 5: Create OPNsense VM (don't start yet)

In Proxmox 1 web UI (https://192.168.1.78:8006):

```
Create VM:
  General:
    Name: opnsense
    VM ID: (next available)
  OS:
    ISO: OPNsense-24.7-dvd-amd64.iso
    Type: Linux 6.x
  System:
    Machine: q35
    BIOS: OVMF (UEFI)
    EFI Disk: yes
  Disks:
    Storage: local-lvm (or your preferred)
    Size: 16 GB
    Discard: yes (if SSD)
  CPU:
    Cores: 2
    Type: host
  Memory:
    Memory: 2048 MB
  Network:
    net0: bridge=vmbr0, vlan=1, model=virtio   ← WAN (home network)
    net1: bridge=vmbr1, vlan=10, model=virtio  ← LAN (homelab)
```

**Do NOT start the VM yet.** Start it after the cable swap.

---

## Step 6: Switch Swap + OPNsense Boot

### 6.1 Shutdown homelab devices

```bash
# From Ubuntu controller, shutdown Pi 1 gracefully
ssh kairos@<pi-ip>
sudo poweroff

# Shutdown Ubuntu controller
sudo poweroff
```

### 6.2 Swap switches

```
Physical cable changes:

Managed switch:
  Port 1 ← extender (was: Mercury Port 1)
  Port 2 ← Proxmox 1 (was: Mercury Port X)
  Port 3 ← Pi 1 (was: Mercury Port Y)
  Port 4 ← Ubuntu (was: Mercury Port Z)
  Port 5 ← Mercury switch uplink (Mercury still has Proxmox 2,3,4)
```

### 6.3 Apply new network config on Proxmox 1

Proxmox 1 is now on the trunk port. Apply the vmbr1 config:

```bash
# SSH to Proxmox 1 (still reachable at 192.168.1.78 via VLAN 1 on trunk)
ssh root@192.168.1.78

# Apply the new bridge config
ifreload -a

# Verify
ip addr show vmbr0    # should have 192.168.1.78
ip addr show vmbr1    # should have 10.0.0.78
```

If `ifreload` is not available:
```bash
systemctl restart networking
```

### 6.4 Start OPNsense VM

```
Proxmox UI → opnsense VM → Start
Open VM console (Proxmox UI → Console)
```

### 6.5 OPNsense initial setup (via VM console)

At the OPNsense console (in Proxmox UI):

```
1. Login: installer / opnsense
2. Install OPNsense to disk (accept defaults)
3. After install, reboot (remove ISO)
4. At console, login: root / opnsense
5. Assign interfaces:
   WAN  → vtnet0 (first NIC, connected to vmbr0/VLAN 1)
   LAN  → vtnet1 (second NIC, connected to vmbr1/VLAN 10)
6. Set LAN IP: 10.0.0.1/24
7. WAN: DHCP (gets 192.168.1.x from home router)
```

---

## Step 7: OPNsense Web UI Configuration

From Proxmox 1 (which has 10.0.0.78 on vmbr1):

```bash
# SSH tunnel from Mac through Proxmox 1:
ssh -L 8443:10.0.0.1:443 root@192.168.1.78
# Then open https://localhost:8443 on Mac
```

### 7.1 Initial setup wizard

```
Login: root / opnsense
Run wizard:
  ├── Hostname: opnsense
  ├── Domain: homelab.local
  ├── Primary DNS: 8.8.8.8
  ├── Secondary DNS: 1.1.1.1
  └── WAN: DHCP (already configured)
```

### 7.2 Configure DHCP server on LAN

```
Services → DHCPv4 → LAN:
  ├── Enable: yes
  ├── Range: 10.0.0.100 - 10.0.0.200
  ├── DNS servers: 10.0.0.1, 8.8.8.8
  └── Gateway: 10.0.0.1
```

### 7.3 Add DHCP static mappings

```
Services → DHCPv4 → LAN → Static Mappings:

  +Add (Pi 1):
    MAC: d8:3a:dd:fc:b7:48
    IP: 10.0.0.34
    Hostname: kairos-pi1
    Description: Pi 1 k3s server

  +Add (Ubuntu controller):
    MAC: (run `ip link show` on Ubuntu to find MAC)
    IP: 10.0.0.32
    Hostname: ubuntu-controller
    Description: Ubuntu controller
```

### 7.4 Configure firewall rules

```
Firewall → Rules → WAN:

  +Add (allow SSH from home to homelab):
    Action: Pass
    Interface: WAN
    Protocol: TCP
    Source: 192.168.1.0/24
    Destination: 10.0.0.0/24
    Destination port: 22 (SSH)
    Description: Allow SSH from home to homelab

  +Add (allow OPNsense UI from home):
    Action: Pass
    Interface: WAN
    Protocol: TCP
    Source: 192.168.1.0/24
    Destination: 10.0.0.1
    Destination port: 443 (HTTPS)
    Description: Allow OPNsense UI from home network
```

### 7.5 Administration settings

```
System → Settings → Administration:
  ├── Web GUI port: 443
  └── (optional) restrict to homelab subnet only
```

---

## Step 8: Power On Homelab Devices

```bash
# Pi 1 — power on
# Sends DHCP request on VLAN 10 (Port 3)
# OPNsense sees MAC d8:3a:dd:fc:b7:48 → assigns 10.0.0.34

# Ubuntu — power on
# Sends DHCP request on VLAN 10 (Port 4)
# OPNsense sees MAC → assigns 10.0.0.32
```

---

## Step 9: Verify

```bash
# From Ubuntu controller (10.0.0.32):
ssh kairos@10.0.0.34                    # SSH to Pi — should work
ping 10.0.0.1                           # OPNsense — should work
ping 8.8.8.8                            # Internet — should work (routed through OPNsense)
ping 192.168.1.78                       # Proxmox 1 — should work (routed to VLAN 1)

# From Mac (192.168.1.6):
ssh -i ~/.ssh/id_ed25519 kairos@10.0.0.34   # Should work (WAN firewall rule)
https://10.0.0.1                             # OPNsense UI (WAN firewall rule)

# From Proxmox 1 (192.168.1.78):
ssh kairos@10.0.0.34                    # Should work (Proxmox has both VLANs)

# Verify Proxmox 2,3,4 unchanged:
ssh root@192.168.1.x                    # Each Proxmox host — should work as before
```

---

## Step 10: Post-Migration Cleanup

### 10.1 Remove Option C hack from Pi

If the systemd service (Option C) was installed on the Pi for static IP:

```bash
ssh kairos@10.0.0.34
sudo systemctl disable kairos-static-ip.service
sudo rm /usr/local/lib/systemd/system/kairos-static-ip.service
sudo rm /usr/local/bin/set-static-ip.sh
sudo reboot
# Pi comes back at 10.0.0.34 from OPNsense DHCP — permanent, no hack
```

### 10.2 Update k3s repo for new IP

```bash
cd /Users/Shashank.Pai/Proxmox/kairos-test

# Update all references from 192.168.1.34 to 10.0.0.34:
#   - docs/phase-1-flashing.md
#   - build/cloud-config-pi1.yaml
#   - cloud-config/20_k8s_workloads.yaml (if it references the IP)

git add -A
git commit -m "Migrate to homelab subnet 10.0.0.x (OPNsense)"
git push origin main
```

Pi self-updates on next reboot via git-pull.

### 10.3 Set up OPNsense backup

```
OPNsense UI → System → Configuration → Backups:
  ├── Enable automatic backups
  ├── Schedule: weekly
  └── Destination: encrypted backup to Proxmox 1 storage
```

---

## Rollback Plan

### Quick rollback (5 minutes)

```
1. Unplug everything from managed switch
2. Plug everything back into Mercury switch (original config)
3. Remove vmbr1 from Proxmox 1:
   ssh root@192.168.1.78
   # Remove vmbr1 block from /etc/network/interfaces
   ifreload -a
4. Stop OPNsense VM
5. Everything is back to original state
```

### If Proxmox 1 becomes unreachable after cable swap

```
1. Connect monitor + keyboard to Proxmox 1
2. Login at console
3. Check: ip addr show vmbr0
4. If vmbr0 has no IP, the trunk port config is wrong
5. Move Proxmox 1 cable to Port 1 (VLAN 1 only, not trunk)
6. Reboot Proxmox 1
7. It should come back at 192.168.1.78
8. Fix switch trunk port config and retry
```

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Switch VLAN config wrong | Medium | Proxmox 1 unreachable | Console access, rollback plan |
| OPNsense install fails | Low | No homelab DHCP | Retry install, Pi stays on Option C |
| Existing VMs affected | Very Low | VMs lose network | They're on vmbr0/VLAN 1 — unchanged |
| OPNsense VM crashes later | Low | Homelab loses internet | Reboot VM, set up HA later |
| Mercury switch port count | Low | Can't add more home devices | Use spare managed switch ports |

---

## Future Expansion

### Adding more homelab devices (10.0.0.x)

Only Ports 3 and 4 are on VLAN 10. If you need more homelab ports:

**Option A — Add a small unmanaged switch on Port 3 or 4:**
```
Managed Port 3 (VLAN 10) → 5-port unmanaged switch (~₹500)
  ├── Pi 1 (10.0.0.34)
  ├── Pi 2 (10.0.0.35)
  └── future devices
```

**Option B — Upgrade to 8-port managed switch later (~₹3,500):**
```
8-port managed switch:
  Port 1: VLAN 1 → extender
  Port 2: Trunk → Proxmox 1
  Port 3-7: VLAN 10 → homelab devices (5 ports)
  Port 8: VLAN 1 → Mercury switch
```

### Adding more Proxmox hosts to homelab

Move a Proxmox host from Mercury (VLAN 1) to the managed switch trunk port (Port 2).
But Port 2 is taken by Proxmox 1. To add a second Proxmox host on both VLANs, you need
the 8-port switch (which has more trunk-capable ports).

Alternatively, put the new Proxmox host on VLAN 10 only (Port 3 or 4) — it'll be homelab
only, no access to 192.168.1.x VMs. Fine if it's a new host with no existing VMs.

### OPNsense HA (high availability)

With 4 Proxmox hosts, set up a second OPNsense VM on Proxmox 2 for failover:

```
Proxmox 1: OPNsense primary (10.0.0.1)
Proxmox 2: OPNsense secondary (10.0.0.2)
CARP virtual IP: 10.0.0.1 (floats between primary and secondary)
```

Requires Proxmox 2 on a trunk port — needs the 8-port switch upgrade.

---

## Timeline

| Phase | When | Duration | Depends on |
|-------|------|----------|------------|
| Phase 0: Pi fix (Option C) | Today | 30 min | Nothing |
| Phase 1: Order switch | Today | 2 min | Nothing |
| Phase 2: Switch setup + VLAN config | When switch arrives | 30 min | Switch in hand |
| Phase 3: OPNsense VM + migration | When switch arrives | 1-2 hours | Phase 2 complete |
| Phase 4: Cleanup + verify | After migration | 30 min | Phase 3 complete |
