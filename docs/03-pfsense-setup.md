# pfSense Setup

pfSense firewall installation and configuration with VLAN tagging for multi-VLAN routing.

## Prerequisites

* Proxmox VE with configured bridges
* pfSense ISO image uploaded to Proxmox
* Understanding of VLAN tagging concepts
* Understanding of firewall concepts

## Important: Network Architecture

This setup uses **VLAN tagging** on a single physical interface (vtnet1), NOT multiple separate physical interfaces:

```
vtnet0 = WAN (physical)
vtnet1 = LAN base (physical) + VLAN tags:
  - Untagged traffic = VLAN 10 (LAN)
  - Tagged .20 = VLAN 20 (GOAD)
  - Tagged .30 = VLAN 30 (Kali)
  - Tagged .99 = VLAN 99 (WiFi)
```

## Download pfSense ISO

Download latest pfSense CE ISO:

```bash
# From Proxmox shell
cd /var/lib/vz/template/iso/
wget https://atxfiles.netgate.com/mirror/downloads/pfSense-CE-2.7.2-RELEASE-amd64.iso.gz
gunzip pfSense-CE-2.7.2-RELEASE-amd64.iso.gz
```

Or upload via web interface:

```
Storage (local) > ISO Images > Upload
```

## Create VM

### VM configuration

```
General:
  VM ID: 100
  Name: pfSense

OS:
  ISO: pfSense-CE-2.7.2-RELEASE-amd64.iso
  Type: Other
  
System:
  BIOS: Default (SeaBIOS)
  Machine: Default (i440fx)
  
Disks:
  Size: 16GB
  Storage: local-lvm
  
CPU:
  Cores: 2
  Type: host
  
Memory:
  RAM: 2048 MB
  
Network:
  Will be configured manually (see below)
```

**Note:** Do not add network interfaces during VM creation. Add them manually for proper ordering.

### Add network interfaces

Navigate to VM > Hardware > Add > Network Device.

**CRITICAL:** Only add **2 interfaces** - pfSense will use VLAN tagging on vtnet1:

```
net0: vmbr0 (WAN)
  Model: VirtIO
  Comment: WAN - Internet uplink
  
net1: vmbr1 (LAN + VLANs)
  Model: VirtIO
  Comment: LAN base + VLAN trunk
```

**Do NOT add net2, net3, net4!** VLANs will be configured as subinterfaces of vtnet1.

## Install pfSense

Start VM and open console:

```
VM > Console
```

### Installation wizard

**Welcome screen:**

```
Select: Install
```

**Keymap selection:**

```
Select keymap: US (or your preference)
Continue with keymap: Accept
```

**Partitioning:**

```
Select: Auto (UFS) - Guided Disk Setup
```

Wait for installation (5-10 minutes).

**Complete installation:**

```
Select: Reboot
```

Remove ISO from VM:

```
VM > Hardware > CD/DVD Drive > Do not use any media
```

## Initial configuration

### Interface assignment

After reboot, console will show interface assignment wizard:

```
Should VLANs be set up now? n (we'll do this later via web GUI)

Enter WAN interface: vtnet0
Enter LAN interface: vtnet1

Enter Optional interfaces (or press Enter): 
  (press Enter - no OPT interfaces yet)

Proceed? y
```

**Important:** At this stage, we only assign the physical interfaces. VLANs come later.

### Configure WAN

From console menu:

```
Select: 2) Set interface(s) IP address

Enter interface number (1 for WAN): 1

Configure IPv4 via DHCP? y

Configure IPv6 via DHCP6? n

Revert to HTTP as webConfigurator protocol? n
```

Wait for DHCP lease. Note the WAN IP address shown.

### Configure LAN

```
Select: 2) Set interface(s) IP address

Enter interface number (2 for LAN): 2

Configure IPv4 via DHCP? n

Enter IPv4 address: 192.168.10.1
Enter subnet mask: 24

Configure IPv4 upstream gateway? n

Configure IPv6 via DHCP6? n

Enable DHCP on LAN? y
  Start address: 192.168.10.100
  End address: 192.168.10.200

Revert to HTTP? n
```

## Web interface configuration

### Access web interface

From browser on VLAN 10 network (192.168.10.0/24):

```
https://192.168.10.1
```

Default credentials:

```
Username: admin
Password: pfsense
```

**Warning:** Browser will show certificate warning. Accept and proceed.

### Initial setup wizard

**Step 1 - Welcome:**

Click Next.

**Step 2 - Netgate Global Support:**

Uncheck both options (if not purchasing support). Click Next.

**Step 3 - General Information:**

```
Hostname: pfsense
Domain: homelab.local
Primary DNS: 8.8.8.8
Secondary DNS: 1.1.1.1
Uncheck: Override DNS
```

Click Next.

**Step 4 - Time Server:**

```
Timezone: America/Sao_Paulo (or your timezone)
Time server: pool.ntp.org
```

Click Next.

**Step 5 - WAN Configuration:**

```
Type: DHCP (if using ISP DHCP)
RFC1918 Networks: Uncheck "Block RFC1918 Private Networks"
```

**Important:** Unblocking RFC1918 allows WAN to have private IP if ISP router uses NAT.

Click Next.

**Step 6 - LAN Configuration:**

```
LAN IP: 192.168.10.1
Subnet mask: 24
```

Click Next.

**Step 7 - Admin Password:**

```
New password: <strong_password>
Confirm: <strong_password>
```

Click Next.

**Step 8 - Reload:**

Click Reload. Wait for configuration to apply.

**Step 9 - Finish:**

Click Finish.

## Configure VLANs

Now we configure VLAN tagging on the LAN interface.

### Create VLANs

Navigate to:

```
Interfaces > Assignments > VLANs
```

Click **Add** to create each VLAN:

**VLAN 20 (GOAD Lab):**

```
Parent Interface: vtnet1 (LAN)
VLAN Tag: 20
VLAN Priority: (leave empty)
Description: GOAD_TARGETS
```

Click Save.

**VLAN 30 (Kali Attack):**

```
Parent Interface: vtnet1 (LAN)
VLAN Tag: 30
VLAN Priority: (leave empty)
Description: KALI_ATTACK
```

Click Save.

**VLAN 99 (WiFi/Quarentena):**

```
Parent Interface: vtnet1 (LAN)
VLAN Tag: 99
VLAN Priority: (leave empty)
Description: WIFI_QUARENTENA
```

Click Save.

**Result:** You should now see:
- vtnet1.20 (GOAD_TARGETS)
- vtnet1.30 (KALI_ATTACK)
- vtnet1.99 (WIFI_QUARENTENA)

### Assign VLAN interfaces

Navigate to:

```
Interfaces > Assignments
```

Click **Add** next to each VLAN to create interface assignments:

```
Available network ports:
- vtnet1.20 (GOAD_TARGETS) → Click [+ Add]
- vtnet1.30 (KALI_ATTACK) → Click [+ Add]
- vtnet1.99 (WIFI_QUARENTENA) → Click [+ Add]
```

Click **Save**.

**Result:** New interfaces appear as OPT1, OPT2, OPT3.

### Rename and configure interfaces

**OPT1 → VLAN20_TARGETS (GOAD):**

Navigate to: `Interfaces > OPT1`

```
Enable: ☑ Enable interface
Description: VLAN20_TARGETS
IPv4 Configuration Type: Static IPv4
IPv4 Address: 192.168.20.1 / 24
```

Click Save, then Apply Changes.

**OPT2 → VLAN30_ATTACK (Kali):**

Navigate to: `Interfaces > OPT2`

```
Enable: ☑ Enable interface
Description: VLAN30_ATTACK
IPv4 Configuration Type: Static IPv4
IPv4 Address: 192.168.30.1 / 24
```

Click Save, then Apply Changes.

**OPT3 → VLAN99_QUARENTENA (WiFi):**

Navigate to: `Interfaces > OPT3`

```
Enable: ☑ Enable interface
Description: VLAN99_QUARENTENA
IPv4 Configuration Type: Static IPv4
IPv4 Address: 192.168.99.1 / 24
```

Click Save, then Apply Changes.

## Configure DHCP (Optional)

If you want DHCP on VLANs:

### VLAN 20 (GOAD) - NO DHCP

GOAD VMs use static IPs. Skip DHCP configuration.

### VLAN 30 (Kali) - NO DHCP

Kali will use static IP (192.168.30.50). Skip DHCP configuration.

### VLAN 99 (WiFi) - ENABLE DHCP

Navigate to: `Services > DHCP Server > VLAN99_QUARENTENA`

```
☑ Enable DHCP server on VLAN99_QUARENTENA interface

Range:
  From: 192.168.99.100
  To: 192.168.99.200

DNS Servers:
  DNS Server 1: 192.168.99.1 (pfSense):> [!WARNING]
```

Click Save.

## Verification

### Check interface status

```
Status > Interfaces
```

All interfaces should show "up" status with correct IPs:

```
WAN: DHCP from ISP (up)
LAN: 192.168.10.1/24 (up)
VLAN20_TARGETS: 192.168.20.1/24 (up)
VLAN30_ATTACK: 192.168.30.1/24 (up)
VLAN99_QUARENTENA: 192.168.99.1/24 (up)
```

### Check VLAN configuration

```
Interfaces > Assignments > VLANs
```

Should show:

```
vtnet1.20 - VLAN 20 on vtnet1 - GOAD_TARGETS
vtnet1.30 - VLAN 30 on vtnet1 - KALI_ATTACK
vtnet1.99 - VLAN 99 on vtnet1 - WIFI_QUARENTENA
```

### Check routing

```
Diagnostics > Routes
```

Should show routes for all configured networks:

```
192.168.10.0/24 → vtnet1 (LAN)
192.168.20.0/24 → vtnet1.20 (VLAN20_TARGETS)
192.168.30.0/24 → vtnet1.30 (VLAN30_ATTACK)
192.168.99.0/24 → vtnet1.99 (VLAN99_QUARENTENA)
default → vtnet0 (WAN)
```

### Test connectivity from pfSense

```
Diagnostics > Ping

Host: 8.8.8.8
Source: WAN address
```

Should receive replies.

## Console verification

From pfSense console, you should see:

```
WAN (wan)             -> vtnet0     -> v4/DHCP4: <ISP_IP>
LAN (lan)             -> vtnet1     -> v4: 192.168.10.1/24
VLAN20_TARGETS (opt2) -> vtnet1.20  -> v4: 192.168.20.1/24
VLAN30_ATTACK (opt3)  -> vtnet1.30  -> v4: 192.168.30.1/24
VLAN99_QUARENTENA     -> vtnet1.99  -> v4: 192.168.99.1/24
```

**This confirms VLAN tagging is working correctly!**

## Important notes about VLAN tagging

### How it works

```
Physical topology:
Proxmox vmbr1 → pfSense vtnet1 → Single physical connection

Logical topology:
vtnet1 (untagged) = VLAN 10 (192.168.10.0/24)
vtnet1.20 (tagged) = VLAN 20 (192.168.20.0/24)
vtnet1.30 (tagged) = VLAN 30 (192.168.30.0/24)
vtnet1.99 (tagged) = VLAN 99 (192.168.99.0/24)
```

### Traffic flow

**Untagged traffic:**
- Arrives on vtnet1 → Goes to LAN (192.168.10.0/24)

**Tagged traffic with VLAN 20:**
- Arrives on vtnet1 with tag 20 → Goes to vtnet1.20 (192.168.20.0/24)

**Tagged traffic with VLAN 30:**
- Arrives on vtnet1 with tag 30 → Goes to vtnet1.30 (192.168.30.0/24)

**Tagged traffic with VLAN 99:**
- Arrives on vtnet1 with tag 99 → Goes to vtnet1.99 (192.168.99.0/24)

### Proxmox bridge mapping

```
Proxmox Side:
- vmbr0 → pfSense vtnet0 (WAN)
- vmbr1 → pfSense vtnet1 (LAN + trunk for all VLANs)
- vmbr2 → GOAD VMs (will send traffic tagged with VLAN 20)
- vmbr3 → Kali VM (will send traffic tagged with VLAN 30)
- vmbr4 → WiFi/Future (will send traffic tagged with VLAN 99)
```

**Note:** Virtual bridges (vmbr2, vmbr3, vmbr4) are internal to Proxmox. They connect VMs to pfSense's VLAN subinterfaces, but they don't need physical ports because all traffic goes through vmbr1 (the trunk).

## Next steps

Proceed to [VLAN and Firewall Rules](04-vlan-firewall-rules.md) to configure traffic filtering and isolation between VLANs.

## Troubleshooting

### Symptom: Cannot access web interface

**Cause:** Wrong IP or browser not on VLAN 10.

**Solution:**

Verify client machine is on VLAN 10 network (192.168.10.0/24):

```bash
ip addr show
```

Ping pfSense LAN IP:

```bash
ping 192.168.10.1
```

If fails, check physical connection to switch.

### Symptom: WAN shows as down

**Cause:** vmbr0 not connected to internet or DHCP not responding.

**Solution:**

From Proxmox, verify vmbr0 bridge has route to internet:

```bash
# Test from Proxmox host
ping -I vmbr0 8.8.8.8
```

Check pfSense WAN settings:

```
Interfaces > WAN
Verify: Type = DHCP
Try: Save and release/renew DHCP
```

### Symptom: VLANs not showing in console

**Cause:** VLANs not created or not assigned.

**Solution:**

1. **Verify VLANs exist:**
```
Interfaces > Assignments > VLANs
Should show vtnet1.20, vtnet1.30, vtnet1.99
```

2. **Verify interfaces assigned:**
```
Interfaces > Assignments
Should show OPT1, OPT2, OPT3 mapped to VLANs
```

3. **Verify interfaces enabled:**
```
Interfaces > VLAN20_TARGETS
Check: ☑ Enable interface
```

### Symptom: VLAN traffic not routing

**Cause:** No firewall rules or missing routes.

**Solution:**

Check routes exist:
```
Diagnostics > Routes
Verify: 192.168.20.0/24, 192.168.30.0/24, 192.168.99.0/24
```

Check firewall rules (will be configured in next guide):
```
Firewall > Rules > [each VLAN interface]
```

### Symptom: Cannot create VLAN on vtnet1

**Cause:** Interface already in use or wrong parent.

**Solution:**

Verify vtnet1 is the LAN interface:
```
Interfaces > Assignments
LAN should be vtnet1
```

If vtnet1 is not available as parent, check:
```
Status > Interfaces
Verify vtnet1 exists and is UP
```

## See also

* [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
* [pfSense VLAN Configuration](https://docs.netgate.com/pfsense/en/latest/vlan/index.html)
* [Previous: Proxmox Network](02-proxmox-network.md)
* [Next: VLAN Firewall Rules](04-vlan-firewall-rules.md)
