# Proxmox VE Installation

Proxmox VE installation on bare metal server with dual NIC configuration.

## Prerequisites

* USB drive (8GB minimum)
* Proxmox VE 8.0+ ISO image
* Server with virtualization support (Intel VT-x or AMD-V)
* Two network interfaces
* Keyboard and monitor for initial setup

## Download ISO

Download latest Proxmox VE ISO:

```bash
wget https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
```

Verify checksum:

```bash
sha256sum proxmox-ve_*.iso
```

## Create bootable USB

### Linux

```bash
# Identify USB device
lsblk

# Write ISO to USB (replace sdX with your device)
sudo dd if=proxmox-ve_*.iso of=/dev/sdX bs=4M status=progress
sudo sync
```

### Windows

Use Rufus or balenaEtcher to write ISO to USB drive.

## Installation process

### Boot from USB

1. Insert USB drive into server
2. Enter BIOS/UEFI (usually F2, F12, or Del key)
3. Set USB as first boot device
4. Save and exit

### Install Proxmox

**Accept EULA:**

Select "I agree" to proceed with installation.

**Target harddisk:**

```
Select installation disk (SSD recommended)
Filesystem: ext4 (default) or ZFS for advanced setups
```

**Warning:** All data on selected disk will be erased.

**Location and timezone:**

```
Country: Brazil
Time zone: America/Sao_Paulo
Keyboard layout: Brazilian (ABNT2) or US
```

**Password and email:**

```
Root password: <strong_password>
Confirm password: <strong_password>
Email: your-email@domain.com
```

**Note:** Root password is used for Proxmox web interface and SSH access.

**Management network:**

```
Management interface: enp5s0 (select WAN interface temporarily)
Hostname (FQDN): pve.local.domain
IP address: Use DHCP initially
Gateway: <router_IP>
DNS server: <router_IP> or 8.8.8.8
```

**Tip:** You will reconfigure network after installation for dual-NIC setup.

Wait for installation to complete (5-10 minutes).

## Post-installation

### First boot

Remove USB drive and reboot. System will boot into Proxmox VE.

### Access web interface

From client machine on same network:

```
https://<proxmox_IP>:8006
```

Login credentials:

```
Username: root
Password: <password_set_during_install>
```

**Warning:** Browser will show SSL certificate warning. This is expected for self-signed certificate. Proceed anyway.

### Update system

Open Proxmox shell (top-right > Shell) and run:

```bash
# Update package lists
apt update

# Upgrade all packages
apt full-upgrade -y

# Reboot if kernel was updated
reboot
```

### Remove subscription notice

Edit file:

```bash
nano /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
```

Find line (around line 500):

```javascript
if (res === null || res === undefined || !res || res
    .data.status.toLowerCase() !== 'active') {
```

Replace with:

```javascript
if (false) {
```

Save (Ctrl+O) and exit (Ctrl+X).

Clear browser cache to see changes.

**Note:** This only removes the notification popup. System remains fully functional without subscription.

### Configure repositories

Disable enterprise repository:

```bash
# Comment out enterprise repo
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
```

Enable no-subscription repository:

```bash
# Add community repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-community.list
```

Update package lists:

```bash
apt update
```

### Enable IOMMU (optional)

Required for PCI passthrough. Edit GRUB configuration:

```bash
nano /etc/default/grub
```

Find line:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```

Replace with (Intel):

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

Or (AMD):

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

Update GRUB:

```bash
update-grub
```

Load VFIO modules:

```bash
echo "vfio" >> /etc/modules
echo "vfio_iommu_type1" >> /etc/modules
echo "vfio_pci" >> /etc/modules
echo "vfio_virqfd" >> /etc/modules
```

Apply changes:

```bash
update-initramfs -u -k all
reboot
```

Verify IOMMU:

```bash
dmesg | grep -e DMAR -e IOMMU
```

Should show IOMMU enabled.

## Verification

### Check Proxmox version

```bash
pveversion
```

Expected output:

```
pve-manager/8.x.x/xxxxxxxxx (running kernel: 6.x.x-x-pve)
```

### Check network connectivity

```bash
ping -c 4 8.8.8.8
```

### Check system resources

```bash
# CPU info
lscpu | grep -E 'Model name|Socket|Core|Thread'

# Memory
free -h

# Disk space
df -h
```

### List network interfaces

```bash
ip link show
```

Should show at least two interfaces (enp5s0 and enx00e04c6801c8 or similar).

## Next steps

Proceed to [Proxmox Network Configuration](02-proxmox-network.md) to set up dual-NIC configuration with VLANs.

## Troubleshooting

### Symptom: Cannot access web interface

**Cause:** Firewall blocking port 8006 or wrong IP address.

**Solution:**

Verify Proxmox is listening:

```bash
ss -tlnp | grep 8006
```

Check IP address:

```bash
ip addr show
```

Temporarily disable firewall:

```bash
systemctl stop pve-firewall
```

### Symptom: Slow web interface

**Cause:** No-subscription repository not enabled or system needs update.

**Solution:**

Configure community repository as shown above, then:

```bash
apt update && apt full-upgrade -y
```

### Symptom: Installation fails on disk selection

**Cause:** Disk has existing partitions or RAID configuration.

**Solution:**

Boot from live Linux USB and wipe disk:

```bash
# WARNING: This destroys all data
wipefs -a /dev/sdX
sgdisk --zap-all /dev/sdX
```

Retry Proxmox installation.

## See also

* [Proxmox VE Installation Guide](https://pve.proxmox.com/wiki/Installation)
* [Proxmox VE Post-Installation](https://pve.proxmox.com/wiki/Package_Repositories)
* [Next: Network Configuration](02-proxmox-network.md)
