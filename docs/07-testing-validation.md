# Testing and Validation

Complete validation tests for homelab functionality and security.

## Prerequisites

* All components deployed (Proxmox, pfSense, GOAD, Kali)
* Access to all VLANs
* Basic understanding of network testing

## Network connectivity tests

### VLAN isolation verification

Test that VLANs are properly isolated according to firewall rules.

**From VLAN 10 (management/home network):**

```bash
# Should succeed
ping -c 4 8.8.8.8
ping -c 4 192.168.10.1

# Should fail (blocked by firewall)
ping -c 2 192.168.20.10  # GOAD network
ping -c 2 192.168.30.50  # Kali network
nc -zv 192.168.20.10 445  # SMB to GOAD
nc -zv 192.168.30.50 22   # SSH to Kali
```

**From VLAN 30 (Kali):**

```bash
# Should succeed
ping -c 4 192.168.20.10   # GOAD DC
nmap -p 445 192.168.20.10  # SMB to GOAD
curl https://google.com    # Internet via HTTP/HTTPS

# Should fail (blocked by firewall)
ping -c 2 192.168.10.1     # Home network gateway
nc -zv 192.168.10.2 8006   # Proxmox web interface
ssh user@192.168.10.100    # Any host on home network
```

**From VLAN 20 (GOAD VMs):**

```powershell
# Should succeed (from any GOAD VM)
ping 192.168.20.10   # Other GOAD VMs

# Should fail (blocked - no gateway)
ping 8.8.8.8         # Internet
ping 192.168.10.1    # Home network
ping 192.168.30.50   # Kali (one-way only)
```

**Expected results:**

All blocked traffic should timeout or show "Destination Host Unreachable."

### Routing verification

**Check Proxmox routing:**

```bash
# From Proxmox shell
ip route show

# Expected routes:
# default via 192.168.10.1 dev vmbr2
# 192.168.10.0/24 dev vmbr2 proto kernel scope link src 192.168.10.2
```

**Check pfSense routing:**

```
Diagnostics > Routes

Expected routes:
- 192.168.10.0/24 → vtnet1 (LAN)
- 192.168.20.0/24 → vtnet1.20 (GOAD)
- 192.168.30.0/24 → vtnet1.30 (Kali)
- 192.168.99.0/24 → vtnet1.99 (WiFi)
- default → vtnet0 (WAN)
```

**Check Kali routing:**

```bash
# From Kali VM
ip route show

# Expected routes:
# default via 192.168.30.1 dev eth0
# 192.168.20.0/24 via 192.168.30.1 dev eth0
# 192.168.30.0/24 dev eth0 proto kernel scope link src 192.168.30.50
```

### DNS resolution tests

**From VLAN 10:**

```bash
nslookup google.com
# Should resolve via 192.168.10.1 or 8.8.8.8
```

**From VLAN 30 (Kali):**

```bash
# External DNS
nslookup google.com
# Should resolve

# GOAD DNS
nslookup sevenkingdoms.local 192.168.20.10
# Should return DC IP (192.168.20.10)

nslookup dc01.sevenkingdoms.local 192.168.20.10
# Should return 192.168.20.10
```

**From VLAN 20 (GOAD):**

```powershell
# From any GOAD VM
nslookup sevenkingdoms.local
# Should return DC IPs

nslookup google.com
# Should fail (no internet)
```

## Active Directory validation

### Domain controller health

**From Kali VM:**

```bash
# Test DC availability
crackmapexec smb 192.168.20.10 192.168.20.11

# Expected output:
# 192.168.20.10 SMB 445 DC01 [*] Windows Server 2019 x64 (name:DC01) (domain:sevenkingdoms.local)
# 192.168.20.11 SMB 445 DC02 [*] Windows Server 2019 x64 (name:DC02) (domain:sevenkingdoms.local)
```

**From DC01 console:**

```powershell
# Check AD services
Get-Service ADWS,KDC,NTDS,DNS | Select-Object Name,Status

# Expected: All services Running

# Check replication
repadmin /replsummary

# Expected: No errors, successful replication between DCs

# List domain controllers
Get-ADDomainController -Filter *

# Expected: DC01 and DC02 listed
```

### Domain accounts validation

**From Kali VM:**

```bash
# Enumerate domain users
crackmapexec smb 192.168.20.10 -u 'guest' -p '' --users

# Expected: Multiple domain users listed (jon.snow, arya.stark, etc.)

# Enumerate domain groups
crackmapexec smb 192.168.20.10 -u 'guest' -p '' --groups

# Expected: Domain Admins, Enterprise Admins, etc.
```

### Kerberos authentication test

**From Kali VM:**

```bash
# Request TGT (requires valid credentials)
impacket-getTGT sevenkingdoms.local/jon.snow:Password123

# Expected: TGT saved to jon.snow.ccache

# List SPNs (Kerberoasting test)
impacket-GetUserSPNs sevenkingdoms.local/jon.snow:Password123 -dc-ip 192.168.20.10

# Expected: Service accounts with SPNs listed
```

### SMB share enumeration

**From Kali VM:**

```bash
# Anonymous enumeration
crackmapexec smb 192.168.20.0/24 --shares

# Expected: SYSVOL, NETLOGON, and other shares visible

# Authenticated enumeration
crackmapexec smb 192.168.20.0/24 -u 'jon.snow' -p 'Password123' --shares

# Expected: Additional shares accessible with auth
```

## Firewall rule validation

### Verify blocked traffic in logs

**Navigate to pfSense:**

```
Status > System Logs > Firewall
```

Generate blocked traffic from Kali:

```bash
# Try to access home network (should be blocked)
ping -c 5 192.168.10.1
```

**Check pfSense logs:**

Should show entries like:

```
block VLAN30_KALI 192.168.30.50:* → 192.168.10.1:* ICMP
```

### Verify allowed traffic

Generate allowed traffic from Kali to GOAD:

```bash
nmap -p 445 192.168.20.10
```

**Check pfSense logs:**

Should show entries like:

```
pass VLAN30_KALI 192.168.30.50:* → 192.168.20.10:445 TCP
```

## Performance tests

### Resource utilization

**Check Proxmox resource usage:**

```bash
# CPU usage
top

# Memory usage
free -h

# Disk I/O
iostat -x 1

# Network traffic
iftop -i vmbr2
```

**Expected:** CPU < 60%, RAM < 80% with all VMs running.

### VM performance

**From each GOAD VM console:**

```powershell
# Check CPU
Get-Counter '\Processor(_Total)\% Processor Time'

# Check memory
Get-Counter '\Memory\Available MBytes'

# Expected: Responsive system, no excessive paging
```

### Network throughput

**Test bandwidth between VLANs:**

From Kali, install iperf3:

```bash
sudo apt install -y iperf3
```

From GOAD VM, download and run iperf3 server:

```powershell
# Download iperf3 for Windows
Invoke-WebRequest -Uri https://iperf.fr/download/windows/iperf-3.1.3-win64.zip -OutFile iperf3.zip
Expand-Archive iperf3.zip
cd iperf3
.\iperf3.exe -s
```

From Kali, test:

```bash
iperf3 -c 192.168.20.10 -t 10
```

**Expected:** Throughput near physical limit (100 Mbps with SF300-24 non-gigabit ports, 1 Gbps on gigabit ports).

## Snapshot and backup validation

### Create test snapshot

```bash
# From Proxmox shell
qm snapshot 101 test-snapshot "Test snapshot validation"

# List snapshots
qm listsnapshot 101

# Expected: test-snapshot listed
```

### Test snapshot rollback

```bash
# Make a change in VM (create a test file)

# Rollback
qm rollback 101 test-snapshot

# Verify change is reverted

# Remove test snapshot
qm delsnapshot 101 test-snapshot
```

### Backup configuration

**Test pfSense backup:**

```
Diagnostics > Backup & Restore
Download configuration XML
```

**Verify backup:**

Open XML file and verify configuration sections present:

```xml
<interfaces>
<firewall>
<nat>
```

## Security validation

### GOAD vulnerability confirmation

Verify GOAD has expected vulnerabilities for testing:

**Kerberoasting:**

```bash
# From Kali
impacket-GetUserSPNs sevenkingdoms.local/jon.snow:Password123 -dc-ip 192.168.20.10 -request

# Expected: Kerberos tickets returned for service accounts
```

**AS-REP Roasting:**

```bash
# From Kali
impacket-GetNPUsers sevenkingdoms.local/ -dc-ip 192.168.20.10 -usersfile users.txt -format hashcat

# Expected: Hashes for accounts with "Do not require Kerberos preauthentication"
```

**SMB Signing disabled:**

```bash
crackmapexec smb 192.168.20.0/24 --gen-relay-list relayable.txt

# Expected: Some hosts without SMB signing listed
```

### Password policy check

**From DC01:**

```powershell
Get-ADDefaultDomainPasswordPolicy

# Expected: Weak policy for lab purposes
# MinPasswordLength: 7 or lower
# PasswordComplexity: False (possibly)
```

### Lateral movement test

**Test Pass-the-Hash (with valid hash):**

```bash
# From Kali (requires compromised hash)
crackmapexec smb 192.168.20.0/24 -u Administrator -H <NTLM_hash> --local-auth

# Expected: Authentication successful on vulnerable systems
```

## Automated test script

Create comprehensive test script:

```bash
cat << 'EOF' > ~/validate-homelab.sh
#!/bin/bash

RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

echo "===== Homelab Validation Test Suite ====="
echo

# Network connectivity
echo "Testing network connectivity..."
ping -c 2 192.168.30.1 > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} Gateway reachable" || \
  echo -e "${RED}✗${NC} Gateway unreachable"

ping -c 2 192.168.20.10 > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} GOAD DC reachable" || \
  echo -e "${RED}✗${NC} GOAD DC unreachable"

timeout 2 ping -c 1 192.168.10.1 > /dev/null 2>&1 && \
  echo -e "${RED}✗${NC} Home network accessible (should be blocked)" || \
  echo -e "${GREEN}✓${NC} Home network blocked"

curl -I -m 5 https://google.com > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} Internet accessible" || \
  echo -e "${RED}✗${NC} Internet unreachable"

# Active Directory
echo
echo "Testing Active Directory..."
crackmapexec smb 192.168.20.10 > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} DC01 SMB accessible" || \
  echo -e "${RED}✗${NC} DC01 SMB failed"

nslookup sevenkingdoms.local 192.168.20.10 > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} DNS resolution working" || \
  echo -e "${RED}✗${NC} DNS resolution failed"

# Tools
echo
echo "Checking tools..."
which crackmapexec > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} CrackMapExec installed" || \
  echo -e "${RED}✗${NC} CrackMapExec missing"

which bloodhound-python > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} BloodHound ingestor installed" || \
  echo -e "${RED}✗${NC} BloodHound ingestor missing"

which impacket-smbclient > /dev/null 2>&1 && \
  echo -e "${GREEN}✓${NC} Impacket installed" || \
  echo -e "${RED}✗${NC} Impacket missing"

echo
echo "===== Validation Complete ====="
EOF

chmod +x ~/validate-homelab.sh
```

**Run validation:**

```bash
~/validate-homelab.sh
```

## Documentation validation

Verify all documentation is accessible:

```bash
# List all documentation files
ls -lh docs/

# Expected files:
# 01-proxmox-installation.md
# 02-proxmox-network.md
# 03-pfsense-setup.md
# 04-vlan-firewall-rules.md
# 05-goad-deployment.md
# 06-kali-setup.md
# 07-testing-validation.md
# TROUBLESHOOTING.md
```

## Success criteria

Lab is fully functional when:

- [ ] All VMs boot and are accessible
- [ ] VLAN isolation works correctly (blocked traffic actually blocked)
- [ ] Kali can reach GOAD but not home network
- [ ] GOAD VMs can communicate internally but have no internet
- [ ] Active Directory is functional with domain join working
- [ ] DNS resolves correctly on all networks
- [ ] Firewall rules log as expected
- [ ] pfSense routing table is complete
- [ ] Pentesting tools are installed and functional
- [ ] Snapshots can be created and restored
- [ ] Performance is acceptable (no excessive CPU/RAM usage)

## Troubleshooting

### Symptom: Validation script shows multiple failures

**Cause:** Fundamental configuration issue.

**Solution:**

Review setup steps in order:

1. Verify Proxmox network configuration: [02-proxmox-network.md](02-proxmox-network.md)
2. Verify pfSense interface assignments: [03-pfsense-setup.md](03-pfsense-setup.md)
3. Verify firewall rules: [04-vlan-firewall-rules.md](04-vlan-firewall-rules.md)

### Symptom: Intermittent connectivity

**Cause:** Network instability or resource exhaustion.

**Solution:**

Check Proxmox resources:

```bash
free -h
top
```

If resources exhausted, shut down non-essential VMs:

```bash
qm stop <VMID>
```

Check for network loops or broadcast storms:

```bash
iftop -i vmbr2
```

## See also

* [Complete Troubleshooting Guide](TROUBLESHOOTING.md)
* [Previous: Kali Setup](06-kali-setup.md)
* [Back to README](../README.md)
