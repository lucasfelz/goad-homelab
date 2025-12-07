# Proxmox Network Configuration

Network bridge configuration for dual-NIC setup with VLAN support via pfSense VLAN tagging.

## Prerequisites

* Proxmox VE installed
* Two physical network interfaces identified
* Basic understanding of Linux networking
* Understanding of VLAN concepts

## Network Architecture Overview

This setup uses **VLAN tagging on pfSense**, not on Proxmox bridges. Proxmox provides simple bridges that connect VMs to pfSense, which handles all VLAN tagging.

```
Physical NICs:
- enp5s0 (onboard):           WAN/Internet → vmbr0
- enx00e04c6801c8 (USB NIC):  LAN/Switch → vmbr1

Virtual bridges (no VLAN tagging on Proxmox):
- vmbr0: WAN - pfSense WAN interface
- vmbr1: LAN - pfSense LAN + trunk (VLAN 10 untagged + VLANs 20,30,99 tagged)
- vmbr2: GOAD VMs - Internal bridge for VLAN 20 traffic
- vmbr3: Kali VM - Internal bridge for VLAN 30 traffic
- vmbr4: WiFi VMs - Internal bridge for VLAN 99 traffic
```

**Important:** VLANs 20, 30, and 99 are implemented as **tagged subinterfaces on pfSense** (vtnet1.20, vtnet1.30, vtnet1.99), not as separate physical connections!

## Identify Network Interfaces

List available interfaces:

```bash
ip link show
```

Expected output:

```
1: lo: <LOOPBACK,UP,LOWER_UP>
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP>
3: enx00e04c6801c8: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

**Note:** Interface names vary by hardware. Use `ip link` output for your specific names.

## Configure Bridges

### Backup Current Configuration

```bash
cp /etc/network/interfaces /etc/network/interfaces.backup
```

### Edit Network Configuration

```bash
nano /etc/network/interfaces
```

Complete configuration:

```conf
# Loopback
auto lo
iface lo inet loopback

# WAN Bridge (Internet uplink for pfSense)
auto vmbr0
iface vmbr0 inet manual
    bridge-ports enp5s0
    bridge-stp off
    bridge-fd 0
    comment WAN Bridge - pfSense WAN interface

# LAN Bridge (Proxmox Management + pfSense LAN/trunk)
auto vmbr1
iface vmbr1 inet static
    address 192.168.10.2/24
    gateway 192.168.10.1
    bridge-ports enx00e04c6801c8
    bridge-stp off
    bridge-fd 0
    comment LAN Bridge - Management + pfSense trunk

# GOAD Bridge (Internal - for VLAN 20 VMs)
auto vmbr2
iface vmbr2 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment GOAD VMs - Connects to pfSense VLAN 20

# Kali Bridge (Internal - for VLAN 30 VM)
auto vmbr3
iface vmbr3 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment Kali VM - Connects to pfSense VLAN 30

# WiFi Bridge (Internal - for VLAN 99 VMs - )
auto vmbr4
iface vmbr4 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment WiFi/IoT VMs - Connects to pfSense VLAN 99
```

### Understanding the Bridge Architecture

**vmbr0 (WAN):**
- Physical port: enp5s0
- Purpose: Connects pfSense WAN to internet via ISP router
- IP: None (managed by pfSense)

**vmbr1 (LAN/Trunk):**
- Physical port: enx00e04c6801c8
- Purpose: Proxmox management + pfSense LAN/trunk
- IP: 192.168.10.2/24 (Proxmox management)
- Also carries: VLAN-tagged traffic for pfSense (VLANs 20, 30, 99)

**vmbr2, vmbr3, vmbr4 (Internal Virtual Bridges):**
- Physical port: **NONE** (internal only)
- Purpose: Connect VMs to pfSense VLAN interfaces
- How they work:
  - VMs connect to these bridges
  - Bridges connect internally to pfSense
  - pfSense handles VLAN tagging on its end
  - Traffic flows: VM → vmbr2 → pfSense vtnet1.20 (VLAN 20)

### Important: How VLAN Tagging Works

```
Traffic Flow Example (Kali VM):

1. Kali VM (vmbr3) sends packet to 192.168.20.10
2. Packet goes through vmbr3 to pfSense
3. pfSense receives on vtnet1.30 (VLAN 30 interface)
4. pfSense routes to vtnet1.20 (VLAN 20 interface)
5. Packet tagged with VLAN 20
6. Packet goes through vmbr2 to GOAD VM
7. GOAD VM receives packet

Key Point: Proxmox bridges are "dumb" - pfSense handles all VLAN logic!
```

### Apply Configuration

Reload network:

```bash
ifreload -a
```

**Caution:** If connected via SSH, this may disconnect you. Reconnect using new IP (192.168.10.2).

Alternatively, reboot for clean application:

```bash
reboot
```

## Verification

### Check Bridge Status

```bash
ip addr show
```

Expected output:

```
vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    (no IP - WAN bridge)

vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.10.2/24 brd 192.168.10.255 scope global vmbr1

vmbr2: <BROADCAST,MULTICAST,UP,LOWER_UP>
    (no IP - internal bridge)

vmbr3: <BROADCAST,MULTICAST,UP,LOWER_UP>
    (no IP - internal bridge)

vmbr4: <BROADCAST,MULTICAST,UP,LOWER_UP>
    (no IP - internal bridge)
```

### Check Routing Table

```bash
ip route show
```

Expected output:

```
default via 192.168.10.1 dev vmbr1
192.168.10.0/24 dev vmbr1 proto kernel scope link src 192.168.10.2
```

**Note:** Only VLAN 10 (192.168.10.0/24) route exists on Proxmox. Other VLANs (20, 30, 99) are routed by pfSense, not Proxmox!

### Test Connectivity

Ping gateway (pfSense):

```bash
ping -c 4 192.168.10.1
```

Ping internet:

```bash
ping -c 4 8.8.8.8
```

### Verify Bridges in Web Interface

Navigate to:

```
Datacenter > Node > Network
```

Should show all bridges (vmbr0-vmbr4) with appropriate configurations:

```
vmbr0: Active, WAN (enp5s0)
vmbr1: Active, LAN (enx00e04c6801c8) - 192.168.10.2/24
vmbr2: Active, GOAD (no ports)
vmbr3: Active, Kali (no ports)
vmbr4: Active, WiFi (no ports)
```

## Configure DNS

Edit DNS settings:

```bash
nano /etc/resolv.conf
```

Add DNS servers:

```conf
nameserver 192.168.10.1
nameserver 8.8.8.8
nameserver 1.1.1.1
```

**Note:** This file may be overwritten. For persistent DNS, add to vmbr1 in `/etc/network/interfaces`:

```bash
nano /etc/network/interfaces
```

Add to vmbr1 section:

```conf
auto vmbr1
iface vmbr1 inet static
    address 192.168.10.2/24
    gateway 192.168.10.1
    bridge-ports enx00e04c6801c8
    bridge-stp off
    bridge-fd 0
    dns-nameservers 192.168.10.1 8.8.8.8
    dns-search homelab.local
    comment LAN Bridge - Management + pfSense trunk
```

## Common Misconceptions

### ❌ WRONG: "Each VLAN needs a separate physical port"

**Reality:** With VLAN tagging, one physical port (vmbr1 → pfSense vtnet1) carries ALL VLANs.

### ❌ WRONG: "Proxmox needs to tag VLAN traffic"

**Reality:** Proxmox bridges are simple L2 switches. pfSense handles all VLAN tagging.

### ❌ WRONG: "vmbr2, vmbr3, vmbr4 need physical ports"

**Reality:** These are internal virtual bridges. They connect VMs to pfSense virtually, not physically.

### ✅ CORRECT: "Traffic flow"

```
GOAD VM (vmbr2) → Internal → pfSense vtnet1.20 (adds VLAN 20 tag)
Kali VM (vmbr3) → Internal → pfSense vtnet1.30 (adds VLAN 30 tag)
WiFi VM (vmbr4) → Internal → pfSense vtnet1.99 (adds VLAN 99 tag)

All tagged traffic → pfSense vtnet1 → vmbr1 → Physical port → Switch
```

## Advanced: Bridge Details

### Why Virtual Bridges Have No Ports

```bash
# Check bridge configuration
brctl show

# Expected output:
bridge name     bridge id               STP enabled     interfaces
vmbr0           8000.aabbccddeeff       no              enp5s0
vmbr1           8000.001122334455       no              enx00e04c6801c8
vmbr2           8000.000000000000       no              
vmbr3           8000.000000000000       no              
vmbr4           8000.000000000000       no
```

Virtual bridges (vmbr2-4) have no physical interfaces because:
1. They're internal software bridges
2. VMs attach to them via virtual TAP interfaces
3. pfSense connects to them internally
4. All physical traffic goes through vmbr1

### Packet Flow Visualization

```
[Kali VM] ─┐
           │
       [vmbr3] ─────┐
                    │
[GOAD VMs] ─┐       │     ┌─ [vmbr0] ─ [enp5s0] ─ Internet
            │       │     │
        [vmbr2] ────┤─────┤
                    │     │
[WiFi VMs] ─┐       │     └─ [vmbr1] ─ [enx00e04c6801c8] ─ Physical Switch
            │       │              ↑
        [vmbr4] ────┘              │
                                   └─ Proxmox Management (192.168.10.2)
             │
             └─────── [pfSense VM]
                      - vtnet0 (WAN) ← vmbr0
                      - vtnet1 (LAN) ← vmbr1
                      - vtnet1.20 (GOAD) ← vmbr2
                      - vtnet1.30 (Kali) ← vmbr3
                      - vtnet1.99 (WiFi) ← vmbr4
```

## Troubleshooting

### Symptom: Lost connectivity after applying config

**Cause:** Syntax error in `/etc/network/interfaces` or wrong interface names.

**Solution:**

Connect via physical keyboard/monitor. Check configuration:

```bash
cat /etc/network/interfaces
```

Restore backup:

```bash
cp /etc/network/interfaces.backup /etc/network/interfaces
ifreload -a
```

Verify interface names match your hardware:

```bash
ip link show
```

### Symptom: Bridge shows DOWN state

**Cause:** Physical interface not connected or wrong `bridge-ports` value.

**Solution:**

For physical bridges (vmbr0, vmbr1):

Check physical cable connection. Verify interface is UP:

```bash
ip link set enx00e04c6801c8 up
```

For virtual bridges (vmbr2, vmbr3, vmbr4):

DOWN state is normal when no VMs are running. They will show UP when VMs connect.

### Symptom: Cannot ping gateway

**Cause:** Wrong gateway IP or network misconfiguration.

**Solution:**

Verify gateway IP (pfSense LAN):

```bash
ip route show
# Should show: default via 192.168.10.1 dev vmbr1
```

Manually add route if missing:

```bash
ip route add default via 192.168.10.1 dev vmbr1
```

Make persistent in `/etc/network/interfaces`:

```conf
auto vmbr1
iface vmbr1 inet static
    address 192.168.10.2/24
    gateway 192.168.10.1
    ...
```

### Symptom: DNS not resolving

**Cause:** DNS servers not configured or unreachable.

**Solution:**

Test DNS:

```bash
nslookup google.com
```

If fails, add DNS to `/etc/network/interfaces`:

```conf
auto vmbr1
iface vmbr1 inet static
    ...
    dns-nameservers 192.168.10.1 8.8.8.8
```

Reload network:

```bash
ifreload -a
```

### Symptom: Virtual bridges not visible in web UI

**Cause:** Bridges not configured with `auto` directive.

**Solution:**

Ensure each bridge has `auto` line in config:

```conf
auto vmbr2
iface vmbr2 inet manual
    ...
```

Reload network configuration:

```bash
ifreload -a
```

### Symptom: VMs cannot reach other VLANs

**Cause:** This is actually correct! Inter-VLAN routing is controlled by pfSense firewall rules.

**Expected behavior:**
- GOAD VMs (vmbr2) cannot reach internet (by design)
- Kali VM (vmbr3) can reach GOAD (if pfSense allows)
- Traffic between VLANs requires pfSense configuration

**Solution:** Configure firewall rules in pfSense (next guide: 04-vlan-firewall-rules.md)

## Advanced Configuration

### Enable Packet Forwarding

Required for routing, but in this setup pfSense handles routing:

```bash
# Check current setting
sysctl net.ipv4.ip_forward

# Enable if needed (usually not necessary for Proxmox host)
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

**Note:** Proxmox host doesn't need to route. pfSense does all routing.

### Configure MTU (if needed)

If experiencing fragmentation issues:

```bash
nano /etc/network/interfaces
```

Add to bridge:

```conf
auto vmbr1
iface vmbr1 inet static
    ...
    mtu 1450
```

### Monitor Bridge Traffic

Real-time monitoring:

```bash
# Install if not present
apt install -y iftop

# Monitor specific bridge
iftop -i vmbr1
```

## Next Steps

Proceed to [pfSense Setup](03-pfsense-setup.md) to configure firewall with VLAN tagging and routing between VLANs.

## Key Takeaways

1. **Proxmox bridges are simple L2 switches** - no VLAN tagging on Proxmox side
2. **pfSense handles all VLAN logic** - tagging, routing, firewalling
3. **Only vmbr0 and vmbr1 have physical ports** - others are internal
4. **One physical connection carries all VLANs** - via VLAN tagging on pfSense
5. **Proxmox only needs access to VLAN 10** - for management

## See Also

* [Proxmox Network Configuration](https://pve.proxmox.com/wiki/Network_Configuration)
* [Linux Network Bridge](https://wiki.archlinux.org/title/Network_bridge)
* [VLAN on Linux](https://wiki.archlinux.org/title/VLAN)
* [Previous: Proxmox Installation](01-proxmox-installation.md)
* [Next: pfSense Setup](03-pfsense-setup.md)
