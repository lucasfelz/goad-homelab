# Troubleshooting Guide

Complete troubleshooting guide for common issues in GOAD Homelab setup.

## Table of Contents

- [Network Issues](#network-issues)
- [Proxmox Issues](#proxmox-issues)
- [pfSense Issues](#pfsense-issues)
- [GOAD Lab Issues](#goad-lab-issues)
- [Kali VM Issues](#kali-vm-issues)
- [Performance Issues](#performance-issues)
- [Emergency Recovery](#emergency-recovery)

---

## Network Issues

### Issue: Proxmox loses connectivity after network changes

**Symptom:**
```bash
ping 8.8.8.8
# No response, connection lost
```

**Cause:** Incorrect bridge configuration or IP address conflicts.

**Solution:**

1. **Connect via physical console** (keyboard + monitor)

2. **Check current configuration:**
```bash
ip addr show
ip route show
```

3. **Verify correct configuration:**
```bash
# Proxmox should be on VLAN 10 (vmbr1)
# IP: 192.168.10.2/24
# Gateway: 192.168.10.1

# If wrong, fix it:
ip addr flush dev vmbr1
ip addr add 192.168.10.2/24 dev vmbr1
ip route add default via 192.168.10.1 dev vmbr1
```

4. **Make persistent:**
```bash
nano /etc/network/interfaces

# Ensure vmbr1 section shows:
auto vmbr1
iface vmbr1 inet static
    address 192.168.10.2/24
    gateway 192.168.10.1
    bridge-ports enx00e04c6801c8
    bridge-stp off
    bridge-fd 0
```

5. **Apply changes:**
```bash
ifreload -a
```

**Prevention:** Always keep Proxmox management on VLAN 10 (vmbr1) which has physical connectivity.

---

### Issue: Cannot ping between VLANs

**Symptom:**
```bash
# From Kali (192.168.30.50)
ping 192.168.20.10
# Destination Host Unreachable
```

**Diagnosis:**

1. **Check pfSense routing:**
```
Diagnostics > Routes
```

Expected routes:
- 192.168.10.0/24 → vtnet1 (LAN)
- 192.168.20.0/24 → vtnet1.20 (GOAD)
- 192.168.30.0/24 → vtnet1.30 (Kali)
- 192.168.99.0/24 → vtnet1.99 (WiFi)

2. **Check firewall rules:**
```
Firewall > Rules > VLAN30_ATTACK

Should have:
✓ PASS: VLAN30 → VLAN20_TARGETS
```

3. **Test from pfSense:**
```
Diagnostics > Ping
Source: VLAN30_ATTACK
Host: 192.168.20.10
```

**Common causes:**
- Firewall rules blocking traffic
- Missing routes in pfSense
- VM on wrong bridge in Proxmox

**Solution:**

Check VM bridge assignment:
```bash
# From Proxmox
qm config <VMID> | grep net

# Kali should show: bridge=vmbr3
# GOAD VMs should show: bridge=vmbr2
```

---

### Issue: VMs have no internet access

**Symptom:**
```bash
ping 8.8.8.8
# No response
```

**Diagnosis by VLAN:**

**VLAN 10 (Home LAN):**
- Should have internet ✓
- Check: Gateway is 192.168.10.1
- Check: pfSense WAN is up

**VLAN 20 (GOAD):**
- Should NOT have internet ✓ (by design - isolated)
- This is correct behavior

**VLAN 30 (Kali):**
- Should have limited internet (HTTP/HTTPS/DNS only)
- Check firewall rules allow ports 80, 443, 53

**VLAN 99 (WiFi):**
- Should have internet ✓
- Check pfSense NAT rules

**Solution for VLAN 30 (Kali):**

```
Firewall > Rules > VLAN30_ATTACK

Verify rules exist:
1. PASS: TCP ports 80, 443 (any destination)
2. PASS: UDP port 53 (any destination)
3. BLOCK: All other traffic
```

Add missing rules if needed.

---

### Issue: DNS not resolving

**Symptom:**
```bash
nslookup google.com
# Server:  192.168.30.1
# Address: 192.168.30.1#53
# ** server can't find google.com: REFUSED
```

**Cause:** DNS not allowed through firewall or wrong DNS server.

**Solution:**

1. **Check firewall allows DNS:**
```
Firewall > Rules > VLAN30_ATTACK
Verify: PASS UDP port 53 to any
```

2. **Test DNS from pfSense:**
```
Diagnostics > DNS Lookup
Hostname: google.com
```

3. **Configure correct DNS in VM:**
```bash
# On Kali
sudo nano /etc/resolv.conf

nameserver 192.168.30.1
nameserver 8.8.8.8
```

4. **Make persistent (Kali):**
```bash
sudo nano /etc/network/interfaces

iface eth0 inet static
    address 192.168.30.50
    netmask 255.255.255.0
    gateway 192.168.30.1
    dns-nameservers 192.168.30.1 8.8.8.8
```

---

## Proxmox Issues

### Issue: Cannot access Proxmox web interface

**Symptom:** Browser shows "Connection refused" or timeout at https://192.168.10.2:8006

**Diagnosis:**

1. **Check Proxmox is on correct IP:**
```bash
# From Proxmox console
ip addr show vmbr1 | grep inet
# Should show: inet 192.168.10.2/24
```

2. **Check web service running:**
```bash
systemctl status pveproxy
```

3. **Check firewall:**
```bash
# If using Proxmox firewall
pve-firewall status
```

**Solution:**

1. **Fix IP if wrong:**
```bash
ip addr add 192.168.10.2/24 dev vmbr1
```

2. **Restart web service:**
```bash
systemctl restart pveproxy
systemctl restart pvedaemon
```

3. **Access from correct network:**
- Client must be on VLAN 10 (192.168.10.0/24)
- Or configure pfSense to allow access from other VLANs

---

### Issue: VM won't start - no space left

**Symptom:**
```
TASK ERROR: can't create image - no space left on device
```

**Diagnosis:**
```bash
df -h
# Check usage of /var/lib/vz
```

**Solution:**

1. **Remove old backups:**
```bash
ls -lh /var/lib/vz/dump/
rm /var/lib/vz/dump/vzdump-qemu-*.vma.zst
```

2. **Remove unused VM disks:**
```bash
# List all disks
pvesm list local-lvm

# Remove unused
lvremove /dev/pve/vm-XXX-disk-X
```

3. **Increase storage:**
```bash
# Extend LVM if possible
lvextend -L +50G /dev/pve/data
```

---

### Issue: Bridge not showing in web interface

**Symptom:** Virtual bridge (vmbr0, vmbr2, vmbr3, vmbr4) not visible in Proxmox GUI.

**Cause:** Missing `auto` directive in network config.

**Solution:**
```bash
nano /etc/network/interfaces

# Each bridge must have 'auto' line:
auto vmbr0
iface vmbr0 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0

# Repeat for all bridges
```

Reload:
```bash
ifreload -a
```

---

## pfSense Issues

### Issue: pfSense loses WAN connectivity

**Symptom:** pfSense shows WAN as down or no IP address.

**Diagnosis:**

1. **Check WAN interface status:**
```
Status > Interfaces > WAN
State: Should show "up"
```

2. **Check DHCP lease:**
```
Status > DHCP Leases
Look for WAN interface
```

**Solution:**

1. **Release and renew DHCP:**
```
Interfaces > WAN
Save (without changes to trigger renewal)
```

2. **From console (option 8):**
```bash
dhclient -r vtnet0  # Release
dhclient vtnet0     # Renew
```

3. **Check cable connection:**
- Verify vmbr0 (Proxmox) connects to ISP router
- Check physical connectivity

---

### Issue: Cannot access pfSense web interface from LAN

**Symptom:** https://192.168.10.1 times out

**Diagnosis:**

1. **Ping gateway:**
```bash
ping 192.168.10.1
```

2. **Check if pfSense is running:**
```bash
# From Proxmox
qm status 100
# Should show: running
```

**Solution:**

1. **Restart pfSense VM:**
```bash
# From Proxmox
qm reboot 100
```

2. **Check firewall rules:**
```
Firewall > Rules > LAN
Verify: Anti-Lockout Rule exists (port 443, 80)
```

3. **Access via console:**
```
VM > Console
Option 11: Restart webConfigurator
```

---

### Issue: VLAN traffic not working

**Symptom:** VMs on virtual VLANs cannot communicate.

**Diagnosis:**

1. **Check VLAN interfaces exist:**
```
Interfaces > Assignments
Should show:
- VLAN20_TARGETS (vtnet1.20)
- VLAN30_ATTACK (vtnet1.30)
- VLAN99_QUARENTENA (vtnet1.99)
```

2. **Check interfaces are enabled:**
```
Interfaces > VLAN20_TARGETS
☑ Enable interface
```

3. **Check VLAN tags are correct:**
```
Interfaces > VLANs
- VLAN 20 on vtnet1
- VLAN 30 on vtnet1
- VLAN 99 on vtnet1
```

**Solution:**

If VLANs don't exist, create them:
```
Interfaces > VLANs > Add

VLAN 20:
  Parent Interface: vtnet1
  VLAN Tag: 20
  Description: GOAD_TARGETS

VLAN 30:
  Parent Interface: vtnet1
  VLAN Tag: 30
  Description: KALI_ATTACK

VLAN 99:
  Parent Interface: vtnet1
  VLAN Tag: 99
  Description: WIFI_QUARENTENA
```

Then assign them:
```
Interfaces > Assignments
Add each VLAN and enable
```

---

### Issue: Firewall rules not working

**Symptom:** Traffic allowed that should be blocked (or vice versa).

**Diagnosis:**

1. **Check rule order:**
```
Firewall > Rules > [Interface]
Remember: First match wins!
```

2. **Check logs:**
```
Status > System Logs > Firewall
Look for blocks/passes
```

3. **Check states:**
```
Diagnostics > States
Clear all states to test fresh
```

**Solution:**

1. **Fix rule order** (drag and drop):
```
Specific rules ABOVE general rules:
1. BLOCK: specific destination
2. PASS: general traffic
```

2. **Clear states and test:**
```
Diagnostics > States > Reset States
```

3. **Enable logging on rules:**
```
Edit rule > Log packets matched by this rule ☑
```

---

## GOAD Lab Issues

### Issue: Domain Controller won't start

**Symptom:** DC VM stuck at boot or very slow.

**Cause:** Insufficient resources or disk corruption.

**Solution:**

1. **Check resources:**
```bash
# From Proxmox
qm config 101  # DC01

# Verify:
memory: 3072
cores: 2
```

2. **Increase RAM if needed:**
```bash
qm set 101 -memory 4096
```

3. **Check disk space:**
```bash
# From VM console
# Windows: Check C: drive has space
```

4. **Restore from snapshot:**
```bash
# From Proxmox
qm listsnapshot 101
qm rollback 101 clean-initial
```

---

### Issue: Domain services not running

**Symptom:** AD DS, DNS, or other services stopped.

**Diagnosis (from DC console):**
```powershell
Get-Service ADWS,KDC,NTDS,DNS | Select Name,Status
```

**Solution:**

1. **Start services:**
```powershell
Start-Service NTDS  # Active Directory
Start-Service DNS   # DNS Server
Start-Service ADWS  # Active Directory Web Services
```

2. **Check DNS configuration:**
```powershell
nslookup sevenkingdoms.local
# Should resolve to DC01/DC02
```

3. **Check replication:**
```powershell
repadmin /replsummary
# Should show no errors
```

4. **Force replication if needed:**
```powershell
repadmin /syncall /AdeP
```

---

### Issue: GOAD VMs can reach internet

**Symptom:** 
```powershell
# From GOAD VM
ping 8.8.8.8
# Reply from 8.8.8.8  # WRONG! Should fail
```

**Cause:** Incorrect firewall rules or VM has wrong gateway.

**Diagnosis:**

1. **Check VM gateway:**
```powershell
ipconfig /all
# Default Gateway should be BLANK
```

2. **Check pfSense rules:**
```
Firewall > Rules > VLAN20_TARGETS
Should have: BLOCK VLAN20 → Any (internet)
```

**Solution:**

1. **Remove gateway from GOAD VMs:**
```powershell
# Remove default gateway
route delete 0.0.0.0
```

2. **Fix pfSense rules:**
```
Add rule:
Action: Block
Source: VLAN20_TARGETS net
Destination: Any
Place ABOVE any allow rules
```

3. **Verify isolation:**
```powershell
ping 8.8.8.8
# Should timeout (correct behavior)
```

---

## Kali VM Issues

### Issue: Kali cannot reach GOAD network

**Symptom:**
```bash
ping 192.168.20.10
# Destination Host Unreachable
```

**Diagnosis:**

1. **Check Kali IP and gateway:**
```bash
ip addr show
# Should be: 192.168.30.50/24

ip route show
# Should include: 192.168.20.0/24 via 192.168.30.1
```

2. **Check VM bridge:**
```bash
# From Proxmox
qm config 200 | grep net
# Should show: bridge=vmbr3
```

**Solution:**

1. **Fix Kali network:**
```bash
sudo ip addr flush dev eth0
sudo ip addr add 192.168.30.50/24 dev eth0
sudo ip route add default via 192.168.30.1
sudo ip route add 192.168.20.0/24 via 192.168.30.1
```

2. **Make persistent:**
```bash
sudo nano /etc/network/interfaces

auto eth0
iface eth0 inet static
    address 192.168.30.50
    netmask 255.255.255.0
    gateway 192.168.30.1
    dns-nameservers 192.168.30.1 8.8.8.8
```

3. **Fix VM bridge if wrong:**
```bash
# From Proxmox
qm set 200 -net0 virtio,bridge=vmbr3
qm reboot 200
```

---

### Issue: Kali tools not working

**Symptom:**
```bash
crackmapexec smb 192.168.20.10
# Error: Connection refused
```

**Cause:** Firewall blocking, wrong network, or services down on target.

**Solution:**

1. **Test basic connectivity:**
```bash
ping 192.168.20.10
nmap -p 445 192.168.20.10
```

2. **Check pfSense allows traffic:**
```
Firewall > Rules > VLAN30_ATTACK
Verify: PASS VLAN30 → VLAN20
```

3. **Check target services:**
```powershell
# On GOAD VM
Get-Service LanmanServer  # SMB service
# Should be: Running
```

---

### Issue: Cannot update Kali packages

**Symptom:**
```bash
sudo apt update
# Failed to fetch...
```

**Cause:** No internet or DNS not working.

**Diagnosis:**

1. **Test connectivity:**
```bash
ping 8.8.8.8        # Should work
curl google.com      # Should work
nslookup google.com  # Should resolve
```

2. **Check firewall:**
```
Firewall > Rules > VLAN30_ATTACK
Verify: PASS TCP 80, 443
Verify: PASS UDP 53
```

**Solution:**

1. **Add DNS rule if missing:**
```
Firewall > Rules > VLAN30_ATTACK > Add

Action: Pass
Protocol: UDP
Source: VLAN30_ATTACK net
Destination: Any
Destination Port: DNS (53)
```

2. **Fix Kali DNS:**
```bash
sudo nano /etc/resolv.conf
nameserver 8.8.8.8
```

---

## Performance Issues

### Issue: High CPU usage on Proxmox

**Diagnosis:**
```bash
top
# Check which VMs are using CPU

qm monitor <VMID>
info cpus
```

**Solution:**

1. **Limit VM CPU:**
```bash
qm set <VMID> -cores 2 -cpulimit 2
```

2. **Stop unused VMs:**
```bash
qm stop <VMID>
```

3. **Check for I/O wait:**
```bash
iostat -x 1
# High %iowait indicates disk bottleneck
```

---

### Issue: VMs very slow

**Symptom:** Windows VMs taking minutes to boot or respond.

**Cause:** Insufficient RAM causing excessive paging.

**Diagnosis:**
```bash
free -h
# Check if swap is heavily used

# Per-VM check:
qm status <VMID>
```

**Solution:**

1. **Add more RAM to host** (hardware upgrade)

2. **Reduce number of running VMs**

3. **Disable Windows services in GOAD:**
```powershell
# Disable Windows Update
Stop-Service wuauserv
Set-Service wuauserv -StartupType Disabled

# Disable Windows Defender (lab only!)
Set-MpPreference -DisableRealtimeMonitoring $true
```

---

### Issue: Network slow between VMs

**Symptom:** Slow file transfers, high latency.

**Diagnosis:**
```bash
# Test bandwidth
iperf3 -s  # On one VM
iperf3 -c <target_IP>  # On another
```

**Solution:**

1. **Check VM network model:**
```bash
qm config <VMID> | grep net
# Should use: model=virtio (not e1000)
```

2. **Fix if wrong:**
```bash
qm set <VMID> -net0 virtio,bridge=vmbr3
```

3. **Check switch performance:**
- SF300-24: 10/100 Mbps on most ports
- Use Gigabit ports for Proxmox

---

## Emergency Recovery

### Complete network failure

**Symptoms:** Cannot access anything, total connectivity loss.

**Recovery:**

1. **Physical console access required**

2. **Reset network config:**
```bash
# Backup current config
cp /etc/network/interfaces /etc/network/interfaces.broken

# Restore working config
cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

auto vmbr1
iface vmbr1 inet static
    address 192.168.10.2/24
    gateway 192.168.10.1
    bridge-ports enx00e04c6801c8
    bridge-stp off
    bridge-fd 0
EOF

ifreload -a
```

3. **Test connectivity:**
```bash
ping 192.168.10.1
ping 8.8.8.8
```

---

### pfSense completely broken

**Recovery:**

1. **Boot pfSense into single user mode**

2. **Reset to factory defaults:**
```
Option 4: Reset to factory defaults
Confirm: yes
```

3. **Reconfigure from scratch:**
- Follow [03-pfsense-setup.md](03-pfsense-setup.md)

4. **Or restore backup:**
```
Diagnostics > Backup & Restore
Upload previous config XML
Restore
```

---

### GOAD VMs corrupted

**Recovery:**

1. **Restore from snapshot:**
```bash
qm listsnapshot <VMID>
qm rollback <VMID> clean-initial
qm start <VMID>
```

2. **Or redeploy GOAD:**
- Delete broken VMs
- Follow [05-goad-deployment.md](05-goad-deployment.md)

---

## Getting Help

If issues persist after troubleshooting:

1. **Check logs:**
```bash
# Proxmox
tail -f /var/log/syslog

# pfSense
Status > System Logs > System
```

2. **Collect diagnostic info:**
```bash
# Proxmox version
pveversion -v

# Network config
cat /etc/network/interfaces
ip addr show
ip route show

# VM configs
qm config <VMID>
```

3. **Community resources:**
- Proxmox Forum: https://forum.proxmox.com/
- pfSense Forum: https://forum.netgate.com/
- GOAD Issues: https://github.com/Orange-Cyberdefense/GOAD/issues

---

## See Also

- [Installation Guides](../README.md#installation-guides)
- [Useful Commands](USEFUL-COMMANDS.md)
- [Network Topology](../README.md#network-topology)

---
