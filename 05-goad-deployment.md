# GOAD Lab Deployment

Game of Active Directory (GOAD) vulnerable lab deployment on Proxmox.

## Prerequisites

* Proxmox VE with configured network bridges
* 30GB+ free RAM (6 VMs x 3-5GB each)
* 300GB+ free disk space
* VLAN 20 configured and isolated from internet
* Basic Active Directory knowledge

## GOAD overview

GOAD provides a vulnerable Active Directory forest for penetration testing practice.

**Environment includes:**

* 2 Domain Controllers (DC01, DC02)
* 2 Member Servers (SRV01, SRV02)
* 2 Workstations (WS01, WS02)
* Multiple domains and trusts
* Common AD misconfigurations
* Vulnerable services and accounts

**Vulnerabilities present:**

* Kerberoasting
* AS-REP Roasting
* Unconstrained delegation
* Constrained delegation
* NTLM relay
* Weak passwords
* Excessive permissions

## Installation methods

Three methods available:

1. **Automated (Terraform + Ansible)** - Recommended
2. **Semi-automated (Manual VM creation + Ansible)**
3. **Manual (Complete manual setup)**

This guide covers Method 1 (automated).

## Download GOAD

Clone GOAD repository:

```bash
cd /opt
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
```

Choose provider:

```bash
cd ad/GOAD/providers/proxmox
```

## Configure Terraform

### Install Terraform

```bash
# Download and install
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform version
```

### Configure Proxmox API access

Create Terraform user in Proxmox:

```bash
# From Proxmox shell
pveum user add terraform@pve
pveum aclmod / -user terraform@pve -role Administrator
pveum user token add terraform@pve terraform-token --privsep=0
```

**Note:** Save the token secret displayed. You will need it.

### Create Terraform variables file

```bash
cd /opt/GOAD/ad/GOAD/providers/proxmox
cp terraform.tfvars.template terraform.tfvars
nano terraform.tfvars
```

Edit configuration:

```hcl
# Proxmox API connection
pm_api_url = "https://192.168.10.2:8006/api2/json"
pm_api_token_id = "terraform@pve!terraform-token"
pm_api_token_secret = "<your_token_secret>"

# Proxmox target node
pm_node_name = "pve"

# Network configuration
pm_bridge = "vmbr3"  # VLAN 20 bridge

# Storage
pm_storage = "local-lvm"

# VM resources
vm_cpu_cores = 2
vm_memory = 3072  # 3GB for DCs, 2GB for others
vm_disk_size = "50G"  # Adjust per VM type

# Windows ISO location
iso_win2019 = "local:iso/Windows_Server_2019.iso"
iso_win10 = "local:iso/Windows_10.iso"
```

**Important:** Adjust resource allocation based on available hardware.

## Prepare Windows ISOs

### Download Windows ISOs

Download evaluation versions:

```bash
cd /var/lib/vz/template/iso/

# Windows Server 2019 (evaluation)
wget https://software-download.microsoft.com/download/pr/17763.737.190906-2324.rs5_release_svc_refresh_SERVER_EVAL_x64FRE_en-us_1.iso
mv 17763.*.iso Windows_Server_2019.iso

# Windows 10 (evaluation)
wget https://software-download.microsoft.com/download/pr/19041.264.200511-0456.vb_release_svc_refresh_CLIENTENTERPRISEEVAL_OEMRET_x64FRE_en-us.iso
mv 19041.*.iso Windows_10.iso
```

**Note:** Microsoft provides 180-day evaluation ISOs. Links change periodically.

### Create VirtIO drivers ISO (optional but recommended)

```bash
cd /var/lib/vz/template/iso/
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win.iso
```

## Deploy GOAD with Terraform

### Initialize Terraform

```bash
cd /opt/GOAD/ad/GOAD/providers/proxmox
terraform init
```

### Plan deployment

```bash
terraform plan
```

Review planned resources. Should show 6 VMs to be created.

### Apply deployment

```bash
terraform apply
```

Type `yes` to confirm.

**Warning:** This creates VMs but does NOT configure Active Directory. That comes next.

Wait for completion (10-15 minutes depending on hardware).

### Verify VMs created

Check in Proxmox web interface:

```
Datacenter > Node > VMs
```

Should see 6 new VMs with GOAD naming.

## Configure Active Directory with Ansible

### Install Ansible

```bash
# Install dependencies
apt update
apt install -y python3 python3-pip sshpass

# Install Ansible
pip3 install ansible pywinrm
```

### Configure Ansible inventory

```bash
cd /opt/GOAD/ansible
cp inventory.yml.template inventory.yml
nano inventory.yml
```

Edit IP addresses to match your VLAN 20 subnet:

```yaml
all:
  children:
    domain_controllers:
      hosts:
        dc01:
          ansible_host: 192.168.20.10
        dc02:
          ansible_host: 192.168.20.11
    servers:
      hosts:
        srv01:
          ansible_host: 192.168.20.20
        srv02:
          ansible_host: 192.168.20.21
    workstations:
      hosts:
        ws01:
          ansible_host: 192.168.20.30
        ws02:
          ansible_host: 192.168.20.31
  vars:
    ansible_user: Administrator
    ansible_password: YourStrongPassword123!
    ansible_connection: winrm
    ansible_winrm_transport: ntlm
    ansible_winrm_server_cert_validation: ignore
```

### Run Ansible playbook

**Warning:** This takes 2-4 hours depending on hardware.

```bash
cd /opt/GOAD/ansible
ansible-playbook -i inventory.yml main.yml
```

**Note:** Playbook will reboot VMs multiple times. This is expected.

Monitor progress. Common tasks:

* Windows updates
* Active Directory installation
* Domain join operations
* User/group creation
* Service configuration
* Vulnerability injection

### Handle errors

If playbook fails, identify failed task and rerun:

```bash
ansible-playbook -i inventory.yml main.yml --start-at-task="<task_name>"
```

Or rerun specific role:

```bash
ansible-playbook -i inventory.yml main.yml --tags="<tag_name>"
```

## Verification

### Check VM accessibility

From Kali VM (VLAN 40):

```bash
# Ping all GOAD VMs
for i in 10 11 20 21 30 31; do
  ping -c 2 192.168.20.$i
done
```

### Check AD services

```bash
# DNS resolution
nslookup sevenkingdoms.local 192.168.20.10

# SMB enumeration
crackmapexec smb 192.168.20.0/24
```

Expected output: All hosts responding with hostname and domain info.

### Check domain accounts

```bash
# List domain users
crackmapexec smb 192.168.20.10 -u 'guest' -p '' --users

# Enumerate shares
crackmapexec smb 192.168.20.0/24 -u 'guest' -p '' --shares
```

Should see multiple domain accounts and shares.

### Access VMs via console

From Proxmox web interface:

```
VM > Console
```

Login with Administrator credentials set in Ansible inventory.

Verify domain membership:

```powershell
# Check domain
(Get-WmiObject Win32_ComputerSystem).Domain

# List domain controllers
nltest /dclist:sevenkingdoms.local
```

## Post-deployment configuration

### Create snapshots

Before any testing, create clean snapshots:

```bash
# From Proxmox shell
for vmid in 101 102 103 104 105 106; do
  qm snapshot $vmid clean-initial "Clean GOAD lab before exploitation"
done
```

**Tip:** Roll back to this snapshot after each pentest session.

### Disable Windows Defender (optional)

Makes exploitation easier for learning:

```powershell
# Run on each VM
Set-MpPreference -DisableRealtimeMonitoring $true
```

### Document credentials

Store credentials securely:

```
Domain: SEVENKINGDOMS.LOCAL / NORTH.SEVENKINGDOMS.LOCAL

Domain Admin: administrator / <password_from_inventory>

Regular users:
- jon.snow / <password>
- arya.stark / <password>
- (many others created by Ansible)

Service accounts:
- svc_sql / <password>
- svc_apache / <password>
```

## Manual deployment (alternative)

If automated deployment fails, VMs can be created manually.

### Create VMs manually

For each VM (example: DC01):

```
General:
  VM ID: 101
  Name: GOAD-DC01

OS:
  ISO: Windows_Server_2019.iso

System:
  BIOS: OVMF (UEFI)
  EFI disk: Yes
  Machine: q35

Disks:
  Size: 50GB
  Interface: VirtIO SCSI

CPU:
  Cores: 2
  Type: host

Memory:
  RAM: 3072 MB

Network:
  Bridge: vmbr3 (VLAN 20)
  Model: VirtIO
```

Repeat for all 6 VMs with appropriate resources.

### Install Windows

Boot each VM and install Windows:

```
Install Windows Server 2019 Standard (Desktop Experience)

Network: Configure static IP
  DC01: 192.168.20.10
  DC02: 192.168.20.11
  SRV01: 192.168.20.20
  SRV02: 192.168.20.21
  WS01: 192.168.20.30
  WS02: 192.168.20.31
  
  Subnet: 255.255.255.0
  Gateway: 192.168.20.1 (leave empty for isolation)
  DNS: 192.168.20.10 (or 192.168.20.11 for DC02)
```

### Configure Active Directory manually

**On DC01:**

```powershell
# Install AD DS
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Create new forest
Install-ADDSForest -DomainName "sevenkingdoms.local" -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Force

# Reboot
Restart-Computer
```

**On DC02:**

```powershell
# Install AD DS
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Join as additional DC
Install-ADDSDomainController -DomainName "sevenkingdoms.local" -Credential (Get-Credential) -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Force
```

Continue with member servers and workstations domain join, user creation, vulnerability configuration...

**Note:** Manual configuration is tedious and error-prone. Automated method strongly recommended.

## Next steps

Proceed to [Kali Linux Setup](06-kali-setup.md) to configure attack platform.

## Troubleshooting

### Symptom: Terraform fails with API connection error

**Cause:** Wrong API token or Proxmox SSL certificate issue.

**Solution:**

Verify API token:

```bash
curl -k -H "Authorization: PVEAPIToken=terraform@pve!terraform-token=<token>" \
  https://192.168.10.2:8006/api2/json/nodes
```

Should return JSON with node info.

Check token permissions:

```bash
pveum user list
pveum user token list terraform@pve
```

### Symptom: Ansible playbook fails on WinRM connection

**Cause:** Windows VMs not accessible or WinRM not enabled.

**Solution:**

Verify VM IP accessibility:

```bash
ping 192.168.20.10
```

Check WinRM from Proxmox:

```bash
nc -zv 192.168.20.10 5985
```

Enable WinRM manually via VM console:

```powershell
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
```

### Symptom: VMs have no network connectivity

**Cause:** VMs not on vmbr3 or pfSense blocking traffic.

**Solution:**

Verify VM network interface:

```bash
qm config 101 | grep net
```

Should show `bridge=vmbr3`.

Check pfSense allows VLAN 20 internal traffic:

```
Firewall > Rules > VLAN20_GOAD
Verify: Pass rule for GOAD net to GOAD net
```

### Symptom: Domain join fails

**Cause:** DNS not resolving or DC not reachable.

**Solution:**

From member VM, test DNS:

```powershell
nslookup sevenkingdoms.local 192.168.20.10
```

Should return DC IP. If fails, manually set DNS:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("192.168.20.10","192.168.20.11")
```

Retry domain join:

```powershell
Add-Computer -DomainName "sevenkingdoms.local" -Credential (Get-Credential) -Restart
```

## See also

* [GOAD Official Repository](https://github.com/Orange-Cyberdefense/GOAD)
* [GOAD Proxmox Guide](https://github.com/Orange-Cyberdefense/GOAD/blob/main/docs/install_with_proxmox.md)
* [Terraform Proxmox Provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)
* [Ansible WinRM Setup](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html)
* [Previous: VLAN Firewall Rules](04-vlan-firewall-rules.md)
* [Next: Kali Linux Setup](06-kali-setup.md)
