# Kali Linux Setup

Kali Linux attack platform configuration on VLAN 30 for GOAD penetration testing.

## Prerequisites

* Proxmox VE with configured bridges
* VLAN 30 configured with internet access (limited)
* GOAD lab deployed on VLAN 20
* Kali Linux ISO

## Download Kali ISO

Download latest Kali installer ISO:

```bash
cd /var/lib/vz/template/iso/
wget https://cdimage.kali.org/kali-2024.1/kali-linux-2024.1-installer-amd64.iso
```

Or live ISO for pre-configured experience:

```bash
wget https://cdimage.kali.org/kali-2024.1/kali-linux-2024.1-live-amd64.iso
```

**Note:** Installer ISO recommended for persistent storage and customization.

## Create VM

### VM configuration

```
General:
  VM ID: 200
  Name: Kali-Offensive

OS:
  ISO: kali-linux-2024.1-installer-amd64.iso
  Type: Linux
  Version: 6.x - 2.6 Kernel

System:
  BIOS: Default (SeaBIOS)
  Machine: q35

Disks:
  Size: 60GB
  Storage: local-lvm
  Interface: VirtIO SCSI
  Cache: Write back

CPU:
  Cores: 4
  Type: host

Memory:
  RAM: 4096 MB

Network:
  Bridge: vmbr0 (VLAN 30)
  Model: VirtIO
```

Start VM:

```bash
qm start 200
```

## Install Kali Linux

Open VM console and proceed with installation.

### Graphical install

Select "Graphical install" from boot menu.

**Language, location, keyboard:**

```
Language: English
Location: United States (or your preference)
Keyboard: US (or your layout)
```

**Network configuration:**

```
Hostname: kali-offensive
Domain: (leave empty or local.domain)
```

**Network will auto-configure via DHCP from pfSense. Cancel and configure static IP:**

```
Configure network manually: Yes
IP address: 192.168.30.50
Netmask: 255.255.255.0
Gateway: 192.168.30.1
DNS: 192.168.30.1 (pfSense will forward)
```

**User accounts:**

```
Root password: <strong_password>
Confirm: <strong_password>

Create user: Yes
Full name: Pentester
Username: pentester
Password: <password>
```

**Disk partitioning:**

```
Method: Guided - use entire disk
Select disk: /dev/sda (VirtIO disk)
Partitioning scheme: All files in one partition
Write changes: Yes
```

**Package manager:**

```
Use network mirror: Yes
Mirror: http.kali.org (default)
HTTP proxy: (leave empty)
```

**Software selection:**

```
[ ] Kali Linux Default
[X] Kali Linux Large (includes most tools)
[X] xfce (lightweight desktop)
```

**Note:** Large metapackage includes Active Directory tools needed for GOAD.

**GRUB boot loader:**

```
Install GRUB: Yes
Device: /dev/sda
```

Wait for installation (20-30 minutes).

**Installation complete:**

```
Remove ISO and reboot
```

From Proxmox:

```
VM > Hardware > CD/DVD Drive > Do not use any media
VM > Console > Reboot
```

## Post-installation configuration

### First login

Login with user credentials created during install.

### Update system

```bash
sudo apt update
sudo apt full-upgrade -y
```

### Install additional tools

```bash
# Active Directory enumeration
sudo apt install -y bloodhound crackmapexec impacket-scripts enum4linux-ng ldapdomaindump

# Network scanning
sudo apt install -y masscan rustscan

# Password cracking
sudo apt install -y hashcat john

# Post-exploitation
sudo apt install -y powershell empire starkiller

# Utilities
sudo apt install -y tmux vim neovim git curl wget
```

### Configure static IP (if not set during install)

Edit network interfaces:

```bash
sudo nano /etc/network/interfaces
```

Add configuration:

```conf
# VLAN 30 interface
auto eth0
iface eth0 inet static
    address 192.168.30.50
    netmask 255.255.255.0
    gateway 192.168.30.1
    dns-nameservers 192.168.30.1 8.8.8.8
```

Apply:

```bash
sudo systemctl restart networking
```

Or use NetworkManager:

```bash
sudo nmcli con add type ethernet con-name vlan30 ifname eth0 ip4 192.168.30.50/24 gw4 192.168.30.1
sudo nmcli con mod vlan30 ipv4.dns "192.168.30.1 8.8.8.8"
sudo nmcli con up vlan30
```

### Verify connectivity

**Test gateway:**

```bash
ping -c 4 192.168.30.1
```

**Test GOAD network:**

```bash
ping -c 4 192.168.20.10
```

**Test internet (should work for HTTP/HTTPS only):**

```bash
curl -I https://google.com
```

**Test blocked access to home network:**

```bash
ping 192.168.10.1
# Should fail or timeout (blocked by firewall)
```

### Install BloodHound

BloodHound requires Neo4j database.

**Install Neo4j:**

```bash
# Add Neo4j repository
wget -O - https://debian.neo4j.com/neotechnology.gpg.key | sudo apt-key add -
echo 'deb https://debian.neo4j.com stable 4.4' | sudo tee /etc/apt/sources.list.d/neo4j.list

sudo apt update
sudo apt install -y neo4j
```

**Configure Neo4j:**

```bash
# Start Neo4j
sudo neo4j start

# Wait for startup (15-30 seconds)
sleep 30

# Set initial password
curl -H "Content-Type: application/json" -X POST -d '{"password":"bloodhound"}' -u neo4j:neo4j http://localhost:7474/user/neo4j/password
```

**Install BloodHound GUI:**

```bash
# Download latest release
wget https://github.com/BloodHoundAD/BloodHound/releases/download/4.3.1/BloodHound-linux-x64.zip

# Extract
unzip BloodHound-linux-x64.zip
sudo mv BloodHound-linux-x64 /opt/BloodHound

# Create desktop shortcut
cat << 'EOF' | sudo tee /usr/share/applications/bloodhound.desktop
[Desktop Entry]
Name=BloodHound
Exec=/opt/BloodHound/BloodHound --no-sandbox
Icon=/opt/BloodHound/resources/app/src/img/icon.ico
Type=Application
Categories=Security;
EOF
```

**Launch BloodHound:**

```bash
/opt/BloodHound/BloodHound --no-sandbox &
```

Login:

```
Database URL: bolt://localhost:7687
Username: neo4j
Password: bloodhound
```

### Configure tool aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# Active Directory
alias cme='crackmapexec'
alias bh='bloodhound-python'
alias impacket-psexec='impacket-psexec -target-ip'

# Network
alias ports='rustscan -a'
alias webenum='gobuster dir -u'

# Common commands
alias ll='ls -lah'
alias ..='cd ..'
alias update='sudo apt update && sudo apt full-upgrade -y'
```

Source:

```bash
source ~/.bashrc
```

### Create project directory structure

```bash
mkdir -p ~/pentest/{recon,enum,exploit,loot,reports}
cd ~/pentest
```

### Install Responder

Responder for LLMNR/NBT-NS poisoning:

```bash
sudo apt install -y responder
```

Or latest from GitHub:

```bash
cd /opt
sudo git clone https://github.com/lgandx/Responder.git
cd Responder
sudo pip3 install -r requirements.txt
```

### Install Covenant C2 (optional)

.NET-based C2 framework:

```bash
# Install .NET SDK
wget https://dot.net/v1/dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 7.0

# Clone Covenant
cd /opt
sudo git clone --recurse-submodules https://github.com/cobbr/Covenant
cd Covenant/Covenant
dotnet build
dotnet run
```

Access at `https://localhost:7443`.

## Configure SSH access (optional)

Enable SSH for remote access:

```bash
# Start SSH service
sudo systemctl start ssh
sudo systemctl enable ssh

# Allow SSH through firewall (if UFW enabled)
sudo ufw allow 22/tcp
```

**Warning:** Only enable if needed. Keep SSH key-based auth only.

Generate SSH keys:

```bash
ssh-keygen -t ed25519 -C "kali-offensive"
```

## Verification

### Tool availability check

```bash
# Verify installations
which crackmapexec bloodhound-python impacket-smbserver nmap responder

# Test CrackMapExec
crackmapexec smb 192.168.20.10

# Test Impacket
impacket-smbclient -h

# Test BloodHound ingestor
bloodhound-python -h
```

### Network connectivity matrix

```bash
# Create test script
cat << 'EOF' > ~/test-connectivity.sh
#!/bin/bash
echo "Testing GOAD network (should succeed):"
ping -c 2 192.168.20.10 && echo "✓ GOAD DC reachable"

echo -e "\nTesting home network (should fail):"
timeout 3 ping -c 2 192.168.10.1 || echo "✓ Home network blocked"

echo -e "\nTesting internet HTTP (should succeed):"
curl -I -m 5 https://google.com && echo "✓ HTTPS working"

echo -e "\nTesting SMB to GOAD (should succeed):"
crackmapexec smb 192.168.20.10 && echo "✓ SMB accessible"
EOF

chmod +x ~/test-connectivity.sh
~/test-connectivity.sh
```

## Create attack workflow scripts

### Reconnaissance script

```bash
cat << 'EOF' > ~/pentest/recon-goad.sh
#!/bin/bash
TARGET="192.168.20.0/24"
OUTPUT_DIR="~/pentest/recon"

mkdir -p $OUTPUT_DIR

echo "[*] Running nmap scan..."
nmap -sV -sC -oA $OUTPUT_DIR/nmap-scan $TARGET

echo "[*] Running CrackMapExec..."
crackmapexec smb $TARGET > $OUTPUT_DIR/cme-smb.txt

echo "[*] Enumerating shares..."
crackmapexec smb $TARGET --shares > $OUTPUT_DIR/shares.txt

echo "[*] Reconnaissance complete. Check $OUTPUT_DIR"
EOF

chmod +x ~/pentest/recon-goad.sh
```

### BloodHound collection script

```bash
cat << 'EOF' > ~/pentest/collect-bloodhound.sh
#!/bin/bash
DOMAIN="sevenkingdoms.local"
DC_IP="192.168.20.10"
OUTPUT_DIR="~/pentest/enum/bloodhound"

mkdir -p $OUTPUT_DIR

echo "[*] Enter domain credentials:"
read -p "Username: " USERNAME
read -sp "Password: " PASSWORD
echo

bloodhound-python -d $DOMAIN -u $USERNAME -p $PASSWORD -ns $DC_IP -c All --zip -o $OUTPUT_DIR

echo "[*] BloodHound data collected. Import into BloodHound GUI."
EOF

chmod +x ~/pentest/collect-bloodhound.sh
```

## Next steps

Proceed to [Testing and Validation](07-testing-validation.md) to verify complete lab functionality.

## Troubleshooting

### Symptom: Cannot reach GOAD network

**Cause:** Missing route or firewall blocking.

**Solution:**

Check routing:

```bash
ip route show
```

Should include route to 192.168.20.0/24 via 192.168.30.1. If missing:

```bash
sudo ip route add 192.168.20.0/24 via 192.168.30.1
```

Make persistent:

```bash
sudo nano /etc/network/interfaces
```

Add to eth0 section:

```conf
    up ip route add 192.168.20.0/24 via 192.168.30.1
```

### Symptom: Internet not working

**Cause:** Gateway not set or pfSense blocking.

**Solution:**

Verify default gateway:

```bash
ip route show default
```

Should show `default via 192.168.30.1`. If missing:

```bash
sudo ip route add default via 192.168.30.1
```

Test DNS:

```bash
nslookup google.com
```

If fails, set DNS manually:

```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

### Symptom: BloodHound cannot connect to Neo4j

**Cause:** Neo4j not running or wrong credentials.

**Solution:**

Check Neo4j status:

```bash
sudo neo4j status
```

If not running:

```bash
sudo neo4j start
```

Reset password:

```bash
curl -H "Content-Type: application/json" -X POST -d '{"password":"bloodhound"}' -u neo4j:neo4j http://localhost:7474/user/neo4j/password
```

### Symptom: CrackMapExec not finding modules

**Cause:** CrackMapExec database not initialized.

**Solution:**

Initialize database:

```bash
crackmapexec smb --help
```

First run initializes database. If issues persist:

```bash
rm -rf ~/.cme
crackmapexec smb --help
```

## See also

* [Kali Documentation](https://www.kali.org/docs/)
* [CrackMapExec Wiki](https://wiki.porchetta.industries/)
* [BloodHound Documentation](https://bloodhound.readthedocs.io/)
* [Impacket Examples](https://github.com/fortra/impacket/tree/master/examples)
* [Previous: GOAD Deployment](05-goad-deployment.md)
* [Next: Testing Validation](07-testing-validation.md)
