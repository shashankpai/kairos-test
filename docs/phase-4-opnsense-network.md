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
                     ├── Proxmox 1 (192.168.1.48)
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

## Original Target Network (Planned — ABANDONED)

> **Note:** The original plan used a VLAN trunk port on the TP-Link SG105E to carry
> both VLAN 1 and VLAN 10 on a single cable to Proxmox 1. This was abandoned after
> testing revealed the SG105E could not reliably handle Proxmox cluster (corosync knet)
> traffic when the switch's 802.1Q VLAN processing was active. See the
> [Troubleshooting](#troubleshooting) section for full details. The actual implementation
> uses a USB Ethernet adapter instead of a trunk port.

```
[ABANDONED PLAN]
Main router (192.168.1.1) → Extender → Managed switch
  Port 2: Trunk (VLAN 1+10) → Proxmox 1   ← THIS DID NOT WORK
```

---

## Actual Network (As Built and Working)

```
Airtel router (192.168.1.1)
  │
  └── WiFi → TP-Link RE200 Range Extender
               │
               └── Mercury 8-port unmanaged switch (VLAN 1 — home network)
                     ├── Proxmox 1 (192.168.1.48) ─── eno1 (built-in NIC)
                     │     │
                     │     └── USB Ethernet adapter (enx00e04c2e6978)
                     │           │
                     │           └── TP-Link SG105E managed switch
                     │                 (802.1Q DISABLED — plain untagged switch)
                     │                 ├── Pi 1 (10.0.0.34)
                     │                 └── (future homelab devices)
                     │
                     ├── Proxmox 2 (192.168.1.87)
                     ├── Proxmox 3 (192.168.1.25)
                     ├── Proxmox 4 (192.168.1.47)
                     └── Ubuntu controller (192.168.1.32) [to move to homelab later]

OPNsense VM (on Proxmox 1):
  vtnet0 (WAN) → vmbr0 → eno1 → Mercury switch → 192.168.1.40 (static)
  vtnet1 (LAN) → vmbr1 → USB NIC → Managed switch → 10.0.0.1

Airtel router static route:
  10.0.0.0/255.255.255.0 → gateway 192.168.1.40
```

### What is unchanged

- Airtel router (`192.168.1.1`) — internet, home DHCP
- Range extender (TP-Link RE200) — WiFi bridge, no routing capability
- Proxmox 1 built-in NIC (`eno1`) — still on Mercury, still `192.168.1.48`
- Proxmox 2, 3, 4 — still on Mercury, still `192.168.1.x`
- All existing VMs on all Proxmox hosts — completely unchanged
- Home devices (phones, laptops, TV) — unchanged

### What changed

- Proxmox 1: got a USB Ethernet adapter as second NIC → `vmbr1` (10.0.0.x)
- OPNsense VM (`9002`) on Proxmox 1: routes between home network and homelab
- Managed switch: repurposed as plain untagged homelab switch (802.1Q disabled)
- Pi 1: moved to managed switch, gets `10.0.0.34` from OPNsense DHCP reservation
- Airtel router: static route `10.0.0.0/24 → 192.168.1.40` for cross-network access

### Key IPs

| Device | Interface | IP | Network |
|--------|-----------|-----|---------|
| Proxmox 1 (management) | eno1 → vmbr0 | `192.168.1.48` | Home |
| Proxmox 1 (homelab) | USB NIC → vmbr1 | `10.0.0.48` | Homelab |
| OPNsense WAN | vtnet0 | `192.168.1.40` (static) | Home |
| OPNsense LAN | vtnet1 | `10.0.0.1` | Homelab |
| Pi 1 | end0 | `10.0.0.34` (DHCP reservation) | Homelab |
| Ubuntu controller | eth0 | `192.168.1.32` | Home (to migrate later) |
| Proxmox 2 | eno1 | `192.168.1.87` | Home |
| Proxmox 3 | eno1 | `192.168.1.25` | Home |
| Proxmox 4 | eno1 | `192.168.1.47` | Home |

### How devices reach the internet

```
Pi (10.0.0.34) → default gateway 10.0.0.1 (OPNsense LAN)
  → OPNsense: NAT masquerade (source NAT: 10.0.0.0/24 → 192.168.1.40)
  → OPNsense WAN → Mercury switch → extender → Airtel router → internet
  ← response: internet → Airtel → extender → Mercury → OPNsense WAN
  ← OPNsense: routes back to 10.0.0.34 via LAN
  ← Pi receives response
```

### How home devices reach homelab

```
Mac/Ubuntu (192.168.1.x) → wants to reach 10.0.0.34
  → sends to Airtel router (default gateway)
  → Airtel router: static route says "10.0.0.0/24 → 192.168.1.40"
  → packet forwarded to OPNsense WAN (192.168.1.40)
  → OPNsense firewall: rule allows 192.168.1.0/24 → 10.0.0.0/24
  → OPNsense LAN → USB NIC → managed switch → Pi
  ← response follows same path in reverse
```

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
| 2 | 1 | VLAN 1 | VLAN 10 | Proxmox 1 (trunk — VLAN 1 native, VLAN 10 tagged) |
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

**Port 2 — Untagged on VLAN 1, Tagged on VLAN 10 (TRUNK)**
- Connected to: Proxmox 1
- This is the special port — it carries BOTH VLANs simultaneously
- VLAN 1 is untagged (native) — Proxmox's `vmbr0` receives it as plain Ethernet
- VLAN 10 is tagged — Proxmox's `vmbr0.10` sub-interface picks it up and feeds `vmbr1`
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

SSH to Proxmox 1 (at 192.168.1.48):

```bash
ssh root@192.168.1.48
```

### 3.1 Find the physical NIC name

```bash
ip link show | grep -E "^[0-9]+: (en|eth)" | grep -v "vmbr"
```

On our Proxmox 1, the physical NIC is `eno1` (enslaved to `vmbr0`).

### 3.2 View the current network config

```bash
cat /etc/network/interfaces
```

Before changes, it should look like:

```
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.48/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

iface wlp6s0 inet manual
```

### 3.3 Add VLAN 10 sub-interface and bridge

We use a **VLAN sub-interface** approach (`vmbr0.10`) instead of directly bridging
the physical NIC. This is the safest method — `vmbr0` stays completely untouched,
existing VMs are unaffected.

```bash
# Backup first
cp /etc/network/interfaces /etc/network/interfaces.backup

# Add VLAN 10 sub-interface and homelab bridge
cat >> /etc/network/interfaces << 'EOF'

# VLAN 10 sub-interface for homelab
auto vmbr0.10
iface vmbr0.10 inet manual

# Homelab bridge (VLAN 10)
auto vmbr1
iface vmbr1 inet static
    address 10.0.0.48/24
    bridge-ports vmbr0.10
    bridge-stp off
    bridge-fd 0
EOF
```

### 3.4 Verify the file looks correct

```bash
cat /etc/network/interfaces
```

Should show the original config plus the new VLAN 10 section at the bottom.

### 3.5 Apply the new config

```bash
ifreload -a
```

If `ifreload` is not available:
```bash
systemctl restart networking
```

### 3.6 Verify both bridges are up

```bash
ip addr show vmbr0 | grep inet
ip addr show vmbr1 | grep inet
```

Expected output:
```
    inet 192.168.1.48/24 scope global vmbr0     ← home network (unchanged)
    inet 10.0.0.48/24 scope global vmbr1        ← homelab network (new)
```

Also verify the Proxmox web UI is still accessible at `https://192.168.1.48:8006`
and existing VMs are still reachable — they should be completely unaffected.

### How the VLAN sub-interface works

```
Physical NIC (eno1) — one cable to switch Port 2 (trunk)
  │
  └── vmbr0 (bridge) — handles ALL traffic from the physical NIC
        │
        ├── Untagged traffic → 192.168.1.48 (home network)
        │   └── Proxmox management, existing VMs, internet — all unchanged
        │
        └── vmbr0.10 (VLAN sub-interface) — filters out ONLY VLAN 10 tagged traffic
              │
              └── vmbr1 (bridge) — 10.0.0.48 (homelab network)
                    └── OPNsense VM's LAN interface connects here
```

**Why this approach is safe:**
- `vmbr0` is untouched — existing VMs don't notice anything changed
- `vmbr0.10` only filters VLAN 10 tagged traffic — doesn't affect untagged/VLAN 1 traffic
- `vmbr1` is a new bridge — only OPNsense and future homelab VMs use it
- One physical cable — no additional hardware needed
- Proxmox is reachable on both networks: `192.168.1.48` (home) and `10.0.0.48` (homelab)

**What OPNsense will use:**
```
OPNsense VM:
  ├── net0 → vmbr0 (WAN) → gets 192.168.1.x from home router DHCP
  └── net1 → vmbr1 (LAN) → 10.0.0.1, serves DHCP to homelab devices
```

### Important: switch Port 2 must have VLAN 1 untagged

For the VLAN sub-interface approach to work, the switch trunk port (Port 2) must
have VLAN 1 as **untagged** (not tagged). This is because `vmbr0` is a standard
bridge that expects untagged traffic. Only VLAN 10 is tagged on Port 2.

```
VLAN 1:  Port 2 = Untagged (native VLAN — vmbr0 handles this)
VLAN 10: Port 2 = Tagged (vmbr0.10 picks this up)
```

If VLAN 1 were tagged on Port 2, Proxmox's `vmbr0` would not understand the tagged
traffic and would lose network access.

---

## Step 4: Download OPNsense ISO

The ISO download from Proxmox 1 failed because DNS was not resolving
(`mirror.opnsense.org` could not be resolved). Internet worked (ping to
8.8.8.8 succeeded), but DNS was broken.

### 4.1 Download on Mac browser

Go to https://opnsense.org/download/ and select:
- **Image type:** DVD image (amd64)
- **Mirror:** any mirror

The download is a `.bz2` compressed file (~1.5GB compressed).

### 4.2 Decompress on Mac

```bash
cd ~/Downloads
bunzip2 OPNsense-26.7-dvd-amd64.iso.bz2
# This produces OPNsense-26.7-dvd-amd64.iso
ls -lh OPNsense-26.7-dvd-amd64.iso
```

### 4.3 Upload to Proxmox 1 via web UI

1. Open `https://192.168.1.48:8006` in Mac browser
2. Select node `pve` in the left panel
3. Click **local** storage
4. Click **ISO Images** tab
5. Click **Upload** button
6. Select the `.iso` file from Downloads
7. Wait for upload to complete

### 4.4 Verify on Proxmox 1

```bash
ls -lh /var/lib/vz/template/iso/OPNsense*.iso
# Should show: /var/lib/vz/template/iso/OPNsense-26.7-dvd-amd64.iso
```

---

## Step 5: Create OPNsense VM

### 5.1 Check available VM IDs

```bash
qm list
```

In our cluster, VM IDs 100, 2002, 300, 9000, 9001 were already taken
(9001 was on a stale node `pve6` that is no longer part of the cluster
but still blocks the ID). We used **9002**.

### 5.2 Create the VM

```bash
qm create 9002 \
  --name opnsense \
  --ostype other \
  --machine q35 \
  --bios ovmf \
  --cpu host \
  --cores 2 \
  --memory 2048 \
  --net0 virtio,bridge=vmbr0 \
  --net1 virtio,bridge=vmbr1 \
  --scsihw virtio-scsi-single \
  --scsi0 local-lvm:16 \
  --cdrom local:iso/OPNsense-26.7-dvd-amd64.iso \
  --boot order=ide2 \
  --efidisk0 local-lvm:1
```

**Notes:**
- `net0 → vmbr0` = WAN (home network, VLAN 1)
- `net1 → vmbr1` = LAN (homelab, VLAN 10)
- 2 cores, 2GB RAM (OPNsense recommends 3GB but 2GB works fine for homelab)
- 16GB disk on local-lvm
- OVMF (UEFI) BIOS with EFI disk

### 5.3 Start the VM

```bash
qm start 9002
```

Open the Proxmox web UI → VM 9002 → **Console**.

---

## Step 6: Install OPNsense (via VM console)

### 6.1 Boot from DVD (live mode)

The VM boots into OPNsense live mode from the DVD. Login at the console:

```
Login: root
Password: opnsense
```

### 6.2 Launch the installer

The console menu does NOT have an "Install" option by default. To launch
the installer, type `8` (Shell) and run:

```bash
/usr/local/sbin/opnsense-installer
```

### 6.3 Installer steps

1. **Choose task:** Install UFS (simpler for a VM, ZFS is overkill here)
2. **RAM warning:** It will warn that 2GB is below the recommended 3GB —
   choose **proceed anyway** (2GB is fine for homelab)
3. **Keyboard layout:** Select your layout (default US is fine)
4. **Disk selection:** Select the **hard disk 16GB** (NOT the DVD/CD).
   It may show as `da0`, `vtbd0`, or `ada0` — pick the ~16GB one.
5. **Confirm:** Yes, proceed with install
6. Wait for installation to complete (~2-3 minutes)

### 6.4 Remove ISO before reboot

When installation completes, **do not reboot yet**. From Proxmox SSH:

```bash
qm stop 9002
qm set 9002 --ide2 none,media=cdrom
qm set 9002 --boot order=scsi0
qm start 9002
```

This removes the ISO and sets the VM to boot from disk.

### 6.5 First boot from disk

Open the console again. OPNsense boots from disk and shows the console
menu with options like:
- Assign interfaces
- Set interface IP address
- Reset root password
- Reset factory defaults
- Shell
- pftop
- Firewall log
- Reload all services
- Update from console
- Restore a backup

---

## Step 7: Assign WAN and LAN interfaces

### 7.1 Assign interfaces (menu option 1)

Type `1` for **Assign interfaces**.

```
Configure LAGGs now? → n
Configure VLANs now? → n

WAN interface: vtnet0    ← connected to vmbr0 (home network)
LAN interface: vtnet1    ← connected to vmbr1 (homelab VLAN 10)
Optional interface 1:    ← press Enter (blank, none needed)
Continue? → y
```

### 7.2 Set LAN IP (menu option 2)

Type `2` for **Set interface IP address**.

```
Which interface to configure? → 1 (LAN)

Configure IPv4 address LAN interface via DHCP? → n
  IPv4 address: 10.0.0.1
  Subnet mask: 24
  Upstream gateway: press Enter (blank — LAN is the gateway itself)

Configure IPv6 address LAN interface via DHCP6? → n
New LAN IPv6 address: n

Enable DHCP server on LAN? → y
  DHCP range start: 10.0.0.100
  DHCP range end: 10.0.0.200

Revert to HTTP? → n (keep HTTPS)
Generate self-signed cert for GUI? → y
Restore GUI access defaults? → y
```

After this, OPNsense shows:
```
You can access the web GUI at https://10.0.0.1
```

---

## Step 8: Access OPNsense Web UI

The Mac is on `192.168.1.x` (home network) and OPNsense is at `10.0.0.1`
(homelab VLAN 10) — they can't reach each other directly. We need an SSH
tunnel through a machine that has access to both networks.

### 8.1 Verify OPNsense is listening

From Proxmox 1 (which has both `192.168.1.48` and `10.0.0.48`):

```bash
nc -zv 10.0.0.1 443
# Should show: 10.0.0.1 443 (https) open

curl -k https://10.0.0.1
# Should return the OPNsense login HTML page
```

### 8.2 SSH tunnel from Mac

Direct SSH from Mac to Proxmox failed with `Permission denied (publickey,password)`
because Proxmox had SSH key-only auth configured. The workaround is a
**double tunnel through the Ubuntu controller**:

**Terminal 1 on Mac** — tunnel to Ubuntu:
```bash
ssh -L 9443:localhost:9443 -N arjun@192.168.1.32
```

**Terminal 2 on Mac** — SSH to Ubuntu, then tunnel to OPNsense via Proxmox:
```bash
ssh arjun@192.168.1.32
ssh -L 9443:10.0.0.1:443 -N root@192.168.1.48
```

**Open in Mac browser:**
```
https://localhost:9443
```

Accept the certificate warning (Advanced → Proceed anyway).

### 8.3 Login

```
Username: root
Password: opnsense
```

---

## Step 9: OPNsense Web UI Configuration

### 9.1 Initial setup wizard

```
Login: root / opnsense
Run wizard:
  ├── Hostname: opnsense
  ├── Domain: homelab.local
  ├── Primary DNS: 8.8.8.8
  ├── Secondary DNS: 1.1.1.1
  └── WAN: DHCP (already configured)
```

### 9.2 Configure DHCP server on LAN

```
Services → DHCPv4 → LAN:
  ├── Enable: yes
  ├── Range: 10.0.0.100 - 10.0.0.200
  ├── DNS servers: 10.0.0.1, 8.8.8.8
  └── Gateway: 10.0.0.1
```

### 9.3 Add DHCP static mappings

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

### 9.4 Configure firewall rules

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

### 9.5 Administration settings

```
System → Settings → Administration:
  ├── Web GUI port: 443
  └── (optional) restrict to homelab subnet only
```

---

## Step 10: Connect devices to the managed switch

### 10.1 Disable 802.1Q on the managed switch

After the trunk port approach failed (see [Troubleshooting](#troubleshooting)), the managed
switch is used as a plain untagged switch. 802.1Q VLAN must be disabled:

1. Open switch UI at `http://192.168.1.250`
2. Go to **802.1Q VLAN Configuration**
3. Select **Disable**
4. Click **Apply**

The switch now behaves like an unmanaged switch — all ports are on the same network.

### 10.2 Add USB Ethernet adapter to Proxmox 1

The USB NIC provides the physical path from Proxmox 1 (and OPNsense's LAN) to the managed
switch without touching the existing `eno1` → Mercury switch connection.

1. Plug a USB 3.0 Ethernet adapter (Realtek r8152 based) into a USB 3 port on Proxmox 1
2. Plug the other end into any port on the managed switch

Verify it's detected:

```bash
# On Proxmox 1
dmesg | grep -i "r8152\|enx"
ip link show | grep enx
```

It will appear as `enxXXXXXXXXXXXX` (MAC-based interface name).

**Fix USB autosuspend** — the adapter may disconnect due to power management:

```bash
# Disable autosuspend immediately (before it disconnects)
echo on > /sys/bus/usb/devices/3-2/power/control

# Or disable for all USB devices
for dev in /sys/bus/usb/devices/*/power/control; do echo on > $dev; done
```

To make it permanent, add to kernel boot parameters:

```bash
nano /etc/default/grub
# Add usbcore.autosuspend=-1 to GRUB_CMDLINE_LINUX_DEFAULT
update-grub
```

### 10.3 Update Proxmox 1 network config for USB NIC

Remove the old VLAN sub-interface approach and replace with the USB NIC:

```bash
# On Proxmox 1
nano /etc/network/interfaces
```

Remove the old blocks:
```
# VLAN 10 sub-interface for homelab   ← REMOVE THIS BLOCK
auto vmbr0.10
iface vmbr0.10 inet manual

# Homelab bridge (VLAN 10)            ← REMOVE THIS BLOCK
auto vmbr1
iface vmbr1 inet static
    address 10.0.0.48/24
    bridge-ports vmbr0.10
    bridge-stp off
    bridge-fd 0
```

Add the new USB NIC blocks (replace `enxXXXXXXXXXXXX` with actual interface name):

```
# USB NIC for homelab network
auto enx00e04c2e6978
iface enx00e04c2e6978 inet manual

# Homelab bridge (10.0.0.x) via USB NIC
auto vmbr1
iface vmbr1 inet static
    address 10.0.0.48/24
    bridge-ports enx00e04c2e6978
    bridge-stp off
    bridge-fd 0
```

Apply:

```bash
# Bring USB NIC up first (ifreload will fail if it's down)
ip link set enx00e04c2e6978 up

ifreload -a

# Verify
ip addr show vmbr1 | grep inet   # should show 10.0.0.48/24
ip link show enx00e04c2e6978      # should show state UP
```

### 10.4 Connect Pi 1 to the managed switch

Plug Pi 1's Ethernet cable into any port on the managed switch (that's not the USB NIC port).

Pi 1 will request a DHCP lease from OPNsense. OPNsense's Kea DHCP server should assign
`10.0.0.34` based on the MAC reservation.

Verify:

```bash
# From Proxmox 1
ping -c 3 10.0.0.34
```

---

## Step 11: OPNsense post-installation configuration

### 11.1 Fix NAT (Source NAT masquerade)

After changing WAN from DHCP to static, the automatic NAT rules may not be generated
correctly. We found that `pfctl -s nat` showed no masquerade rules, which prevented
the Pi from reaching the internet.

**Fix:** Add a manual Source NAT rule.

In OPNsense web UI → **Firewall → NAT → Source NAT**:

1. Change mode to **Manual**
2. Click **+Add:**
   - **Interface:** WAN
   - **Version:** IPv4
   - **Protocol:** any
   - **Source Address:** `10.0.0.0/24`
   - **Destination:** any
   - **Translate Source IP:** `Interface address` (= `192.168.1.40`)
   - **Description:** `Masquerade LAN to WAN`
3. Click **Save** → **Apply Changes**

Verify from Pi:

```bash
sudo ping -c 3 8.8.8.8    # must work after this
```

### 11.2 Set static WAN IP

OPNsense WAN was initially on DHCP and got `192.168.1.40`. We made it static so it
never changes (required for the static route on the Airtel router to work).

In OPNsense web UI → **Interfaces → WAN**:

- **IPv4 Configuration Type:** `Static IPv4`
- **IPv4 address:** `192.168.1.40 / 24`

In **System → Gateways → Configuration**, edit `WAN_GW`:

- **Interface:** WAN
- **Address Family:** IPv4
- **IP Address:** `192.168.1.1`

Save → Apply Changes.

Verify from OPNsense shell:

```bash
# type 8 in console for shell
ping -c 3 8.8.8.8       # OPNsense internet — should work
ping -c 3 192.168.1.1   # Airtel router — should work
netstat -rn | grep default   # should show 192.168.1.1
```

### 11.3 Fix Kea DHCP (not starting automatically)

We found that Kea DHCP was not running even though it was configured. Dnsmasq was
competing with Kea and serving IPs from its own pool (Pi got `10.0.0.173` instead
of the reserved `10.0.0.34`).

**Fix 1: Remove Dnsmasq DHCP range**

In OPNsense web UI → **Services → Dnsmasq DNS & DHCP → DHCP ranges**:

- Delete any range defined for LAN

Dnsmasq continues to run for DNS only. Kea takes over DHCP.

**Fix 2: Start Kea DHCP**

In OPNsense web UI → **System → Diagnostics → Services**:

- Find `kea-dhcp4` and click the restart button

Verify it's running:

```bash
# In OPNsense shell
ps aux | grep kea
```

After Kea starts, reboot the Pi so it gets a fresh DHCP lease — it should receive
`10.0.0.34` this time.

### 11.4 OPNsense firewall rules (complete list)

These are all the firewall rules configured:

**LAN rules (auto-generated — do not modify):**

| Interface | Protocol | Source | Destination | Port | Description |
|-----------|----------|--------|-------------|------|-------------|
| LAN | IPv4 any | LAN network | any | any | Default allow LAN to any |
| LAN | IPv6 any | LAN network | any | any | Default allow LAN IPv6 to any |

**WAN rules (manually added):**

| Interface | Protocol | Source | Destination | Port | Description |
|-----------|----------|--------|-------------|------|-------------|
| WAN | TCP | 192.168.1.0/24 | 10.0.0.0/24 | 22 (SSH) | Allow SSH from home to homelab |
| WAN | TCP/UDP | 192.168.1.0/24 | 10.0.0.1 | 443 (HTTPS) | Allow OPNsense UI from home |
| WAN | any | 192.168.1.0/24 | 10.0.0.0/24 | any | Allow home network to homelab |
| WAN | TCP | 192.168.1.0/24 | WAN address | 443 (HTTPS) | Allow OPNsense UI on WAN IP |

**Why the rules are on WAN (not LAN):**

Traffic from home (192.168.1.x) to homelab (10.0.0.x) enters OPNsense on the WAN
interface. OPNsense evaluates the WAN rules for this traffic. The LAN rules apply
to traffic originating from the homelab side.

### 11.5 OPNsense DHCP static reservation (Pi 1)

In **Services → Kea DHCP → Kea DHCPv4 → Subnets**:

```
Subnet: 10.0.0.0/24
Pool: 10.0.0.100 - 10.0.0.200
```

In **Services → Kea DHCP → Kea DHCPv4 → Reservations**:

```
Subnet:      10.0.0.0/24
MAC address: d8:3a:dd:fc:b7:48
IP address:  10.0.0.34
Hostname:    kairos-pi1
Description: Pi 1 k3s cluster
```

---

## Step 12: Cross-network access from home devices

### 12.1 Static route on Airtel router

For all home network devices (Mac, Ubuntu, phones) to reach 10.0.0.x without tunnels:

1. Open `http://192.168.1.1` (Airtel router)
2. Go to **Advanced → Routing → Static Routing**
3. Click **New Item:**
   - **Name:** `homelab`
   - **Egress:** `LAN`
   - **Network Address:** `10.0.0.0`
   - **Subnet Mask:** `255.255.255.0`
   - **Gateway:** `192.168.1.40`

The router sends ICMP redirects to clients telling them to use `192.168.1.40` for
`10.0.0.x` traffic. This works for all wired and WiFi devices on the home network.

**Note:** The TP-Link RE200 extender does not support static routes. All routing is
handled at the Airtel router level.

### 12.2 Access OPNsense web UI (no tunnel)

With the WAN firewall rule in place, the OPNsense UI is accessible directly:

```
From Mac/Ubuntu: https://192.168.1.40
Login: root / <your password>
```

No SSH tunnel required.

### 12.3 SSH to Pi (no jump host)

With the static route on the Airtel router, direct SSH works from any home device:

```bash
# From Mac or Ubuntu
ssh kairos@10.0.0.34
```

If you need to use SSH agent forwarding (when the key is only on Mac):

```bash
ssh-add ~/.ssh/id_ed25519
ssh -A kairos@10.0.0.34
```

### 12.4 Verify complete setup

Run these checks in order:

```bash
# 1. From Proxmox 1 — verify OPNsense LAN
ping -c 3 10.0.0.1      # should work

# 2. From OPNsense shell (console → type 8) — verify internet
ping -c 3 8.8.8.8       # should work
ping -c 3 192.168.1.1   # should work

# 3. From Pi — verify internet via OPNsense
sudo ping -c 3 8.8.8.8  # should work

# 4. From Ubuntu (192.168.1.32) — verify cross-network access
ping 10.0.0.34           # should work (via Airtel static route)
ssh kairos@10.0.0.34     # should work
nc -zv 192.168.1.40 443  # should connect (OPNsense UI)

# 5. From Mac — add route if needed, then test
sudo route -n add -net 10.0.0.0/24 192.168.1.40  # Mac-specific
ping 10.0.0.34
ssh kairos@10.0.0.34
```

---

## Troubleshooting

### Problem 1: Trunk port approach — managed switch dropped cluster traffic

**What happened:** When Proxmox 1 was moved to the managed switch Port 2 (configured
as trunk: VLAN 1 untagged + VLAN 10 tagged), the Proxmox cluster became unstable.
After 2-5 minutes, the Proxmox web UI would become unreachable and the cluster would
lose quorum visibility.

**Symptoms:**
- Initial connection works fine
- After 2-5 minutes: Proxmox web UI stops loading
- SSH to Proxmox 1 times out
- Moving cable back to Mercury switch immediately restores everything
- `pvecm status` showed `Quorate: Yes` immediately after moving back

**Tests performed:**
| Test | Result |
|------|--------|
| Proxmox 1 on Mercury (unmanaged) | Stable — works perfectly |
| Proxmox 1 on managed Port 2, VLAN 1 untagged + VLAN 10 tagged | Drops after 2-5 min |
| Proxmox 1 on managed Port 2, VLAN 10 removed (VLAN 1 only) | Still drops |
| IGMP snooping disabled on switch, retry | Still drops |
| 802.1Q disabled entirely on switch | Flaky — intermittent drops |

**Root cause:** The TP-Link SG105E is a budget entry-level managed switch. Its internal
switching capacity and packet buffer are insufficient for Proxmox corosync knet traffic,
which sends frequent small UDP packets (port 5405) between all cluster nodes. When the
managed switch was in the path between Proxmox 1 and the rest of the cluster, the switch
couldn't handle the traffic pattern reliably regardless of VLAN settings.

**Resolution:** Kept Proxmox 1 on the Mercury switch (proven stable). Used a USB Ethernet
adapter as a second NIC to connect Proxmox 1 to the managed switch for VLAN 10 only.
The managed switch never sees cluster traffic.

---

### Problem 2: USB NIC disconnecting (error -71)

**Symptoms:**
```
r8152-cfgselector 3-2: USB disconnect, device number 14
usb 3-1: device not accepting address 12, error -71
Cannot find device "enx00e04c2e6978"
```

**Cause:** USB autosuspend — Linux power management puts the USB device to sleep.

**Fix:**
```bash
echo on > /sys/bus/usb/devices/3-2/power/control
```

If the device path isn't found (device already disconnected), replug the adapter and
immediately run:
```bash
for dev in /sys/bus/usb/devices/*/power/control; do echo on > $dev; done
```

---

### Problem 3: Pi getting dynamic IP (10.0.0.173) instead of reserved (10.0.0.34)

**Cause:** Both Dnsmasq and Kea DHCP were enabled and competing. Dnsmasq was winning
and serving IPs from its own pool without honoring Kea's MAC reservations.

**Fix:**
1. Delete the DHCP range in **Services → Dnsmasq → DHCP ranges**
2. Ensure Kea DHCP is enabled and started (**System → Diagnostics → Services**)
3. Reboot Pi to get a fresh DHCP lease

---

### Problem 4: Kea DHCP not starting automatically

**Symptoms:** `ps aux | grep kea` shows nothing. Pi gets no DHCP lease.

**Cause:** Kea requires the subnet to be configured AND the service to be explicitly
saved/applied in the web UI. It doesn't start automatically just from being "enabled".

**Fix:**
1. Go to **Services → Kea DHCP → Kea DHCPv4 → Settings** — verify Enabled + LAN selected
2. Go to **Services → Kea DHCP → Kea DHCPv4 → Subnets** — verify `10.0.0.0/24` exists with pool
3. Go to **System → Diagnostics → Services** → find `kea-dhcp4` → click restart

---

### Problem 5: Pi has internet routing but no actual internet (NAT missing)

**Symptoms:**
- `ping 10.0.0.1` works from Pi (OPNsense reachable)
- `ping 8.8.8.8` fails from Pi
- `pfctl -s nat` on OPNsense shows no masquerade rules
- OPNsense itself can ping 8.8.8.8

**Cause:** After changing WAN from DHCP to static, the automatic NAT rule generation
did not create a masquerade rule. Traffic from Pi reached OPNsense and was forwarded
to the internet, but the source IP was still `10.0.0.34` — the internet couldn't route
the response back to a private address.

**Fix:** Add manual Source NAT rule in **Firewall → NAT → Source NAT**:

```
Mode: Manual
Interface: WAN
Source: 10.0.0.0/24
Destination: any
Translate Source IP: Interface address (192.168.1.40)
```

---

### Problem 6: Home devices can't reach 10.0.0.x

**Symptoms:**
- Airtel router sends ICMP redirect: `Redirect Host(New nexthop: 192.168.1.40)`
- But ping/nc still fails

**Cause (two separate issues):**
1. OPNsense WAN firewall was blocking all incoming traffic from home network to homelab
2. Ubuntu/Mac wasn't accepting the ICMP redirect and updating its local route

**Fix 1 — OPNsense WAN firewall rule:**

In **Firewall → Rules → WAN** add:
```
Action: Pass
Protocol: any
Source: 192.168.1.0/24
Destination: 10.0.0.0/24
Description: Allow home network to homelab
```

**Fix 2 — OPNsense UI accessible on WAN IP:**

In **Firewall → Rules → WAN** add:
```
Action: Pass
Protocol: TCP
Source: 192.168.1.0/24
Destination: WAN address
Port: 443
Description: Allow OPNsense UI from home
```

**Fix 3 — Manual route on Linux if redirect isn't accepted:**

```bash
# Ubuntu
sudo ip route add 10.0.0.0/24 via 192.168.1.40

# Mac
sudo route -n add -net 10.0.0.0/24 192.168.1.40
```

---

## Post-Migration Tasks (Pending)

### Ubuntu controller migration

Ubuntu controller is still on the home network (`192.168.1.32` on Mercury switch).
To move it to the homelab network:

1. Find Ubuntu's MAC address: `ip link show` on Ubuntu
2. Add DHCP reservation in OPNsense Kea: MAC → `10.0.0.32`
3. Move Ubuntu's cable to the managed switch
4. Reboot Ubuntu — it gets `10.0.0.32` from OPNsense

> **Note:** Once Ubuntu is on 10.0.0.x, you can no longer SSH to it from home directly
> without the static route on the Airtel router. Ensure the route is in place first.

### Remove Option C static IP hack from Pi

The Pi may still have the systemd static IP service from the old approach:

```bash
ssh kairos@10.0.0.34
sudo systemctl status kairos-static-ip 2>/dev/null

# If it exists, remove it:
sudo systemctl disable kairos-static-ip.service
sudo rm /usr/local/lib/systemd/system/kairos-static-ip.service
sudo rm /usr/local/bin/set-static-ip.sh
sudo reboot
```

OPNsense's DHCP reservation now handles the static IP permanently.

### Update k3s repo for new IP

```bash
cd /Users/Shashank.Pai/Proxmox/kairos-test

# Update all references from 192.168.1.34 to 10.0.0.34:
#   - docs/phase-1-flashing.md
#   - build/cloud-config-pi1.yaml
#   - cloud-config/20_k8s_workloads.yaml

git add -A
git commit -m "Migrate Pi to homelab subnet 10.0.0.x (OPNsense)"
git push origin main
```

### OPNsense backup

```
OPNsense UI → System → Configuration → Backups:
  ├── Enable automatic backups
  ├── Schedule: weekly
  └── Download a manual backup now as a baseline
```

---

## Rollback Plan

### Quick rollback — restore original flat network

```
1. Move Pi cable back to Mercury switch
2. Remove USB NIC from managed switch
3. Stop OPNsense VM: qm stop 9002
4. Pi will DHCP on 192.168.1.x from Airtel router
5. Everything else is unchanged — Proxmox cluster never moved
```

### Remove vmbr1 from Proxmox 1 (optional cleanup)

```bash
ssh root@192.168.1.48
nano /etc/network/interfaces
# Remove the enxXXXX and vmbr1 blocks
ifreload -a
```

---

## Future Expansion

### Adding more homelab devices

The managed switch has 5 ports. With USB NIC on Port 1 and Pi on Port 2, there are
3 free ports for future homelab devices.

For more capacity, add a small unmanaged switch to the managed switch:

```
Managed switch Port 3 → 5-port unmanaged switch
  ├── Pi 2
  ├── Pi 3
  └── future devices
```

All devices get IPs from OPNsense DHCP (`10.0.0.100-200` pool). Add reservations
for each as needed.

### Upgrade path: dual-NIC mini PC as dedicated firewall

The current setup (OPNsense as a VM on Proxmox 1) works well but has a single point
of failure. The better long-term approach is a **dedicated OPNsense box** with two
built-in NICs:

```
Dedicated OPNsense mini PC (e.g. Protectli, Beelink with dual NIC):
  NIC 1 (WAN) → Mercury switch → 192.168.1.40
  NIC 2 (LAN) → Managed switch → 10.0.0.1

Benefits:
  - OPNsense independent of Proxmox — if Proxmox goes down, homelab still has internet
  - No USB NIC reliability concerns
  - Standard firewall deployment
  - Can be reinstalled in <30 min using this documentation
```

### Inter-VLAN communication

Any device on `192.168.1.x` can reach `10.0.0.x` provided:
1. The Airtel static route is in place (`10.0.0.0/24 → 192.168.1.40`)
2. The OPNsense WAN firewall rule allows the traffic

OPNsense controls all cross-network access via firewall rules — add or restrict
rules in **Firewall → Rules → WAN** as needed.
