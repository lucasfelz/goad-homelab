# Cybersecurity Homelab with GOAD Lab

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Proxmox](https://img.shields.io/badge/Proxmox-VE%208-orange)](https://www.proxmox.com/)
[![pfSense](https://img.shields.io/badge/pfSense-2.7-blue)](https://www.pfsense.org/)
[![GOAD](https://img.shields.io/badge/GOAD-Active%20Directory%20Lab-red)](https://github.com/Orange-Cyberdefense/GOAD)

> **Professional homelab for Cybersecurity, Pentesting, and Active Directory studies with fully isolated GOAD (Game of Active Directory) environment.**

---

## Table of Contents

- [About the Project](#-about-the-project)
- [Architecture](#-architecture)
- [Hardware Requirements](#-hardware-requirements)
- [Network Topology](#-network-topology)
- [Installation Guides](#-installation-guides)
- [Troubleshooting](#-troubleshooting)
- [References](#-references)

---

## About the Project

This project documents the complete implementation of a homelab focused on cybersecurity and pentesting, with the following features:

### Key Features

- **GOAD (Game of Active Directory) Environment** - Lab with 6 vulnerable Windows VMs
- **Complete network isolation** - GOAD without internet access for maximum security
- **Separate offensive network** - Kali Linux on dedicated VLAN
- **pfSense as firewall/router** - Central controller for VLANs
- **Proxmox VE** - Enterprise-grade virtualization
- **SF300-24 Switch** - Managed 24-port 10/100mbps switch + 4 Gigabit ports for tagging

### Learning Objectives

- Pentesting in Active Directory environments
- Windows vulnerability exploitation (Kerberoasting, AS-REP Roasting, etc.)
- Network segmentation and firewalling
- Virtualization and infrastructure management
- Traffic analysis and incident response

---

## Architecture

### Main Components

| Component | Description | Specifications |
|------------|-----------|----------------|
| **Proxmox VE 8.4** | Virtualization host | Xeon E5-2630 v4, 16GB RAM |
| **pfSense** | Virtual Firewall/Router | 5 network interfaces, inter-VLAN routing |
| **GOAD Lab** | 6 vulnerable Windows VMs | Domain Controllers, Servers, Workstations (Work in progress) |
| **Kali Linux** | VM for pentesting | Interface on VLAN 30 (offensive) |
| **Cisco SF300-24** | Managed switch | 24x 10/100 Mbps + 4x Gigabit, VLAN support |

### Implemented VLANs

| VLAN | Name | Network | Description | Internet |
|------|------|------|-----------|----------|
| **1** | WAN | DHCP | Internet via ISP | ✅ |
| **10** | Home LAN | 192.168.10.0/24 | Main network, Proxmox management | ✅ |
| **20** | GOAD Lab | 192.168.20.0/24 | Vulnerable Active Directory environment (ISOLATED) | ❌ |
| **30** | Management | 192.168.30.0/24 | Attack machine, limited access | ⚠️Limited |
| **99** | QUARENTENA (Huawei Access Point) | 192.168.99.0/24 | Network WiFi isolated, limited access | ⚠️ Limited |

---

## Hardware Requirements

### Proxmox Server

```
CPU:     Intel Xeon E5-2630 v4 (10 cores, 20 threads)
RAM:     16GB DDR4 (minimum) - Recommended: 32GB+
Storage: 256GB+ SSD (for GOAD VMs) + HDD for backups
NICs:    2x Ethernet (1x onboard + 1x USB)
         - eth0 (enp5s0): WAN/Internet
         - eth1 (enx00e04c6801c8): LAN/Switch      # I use a adapter USB to Ethernet with 2500mpbs speed
```

### Client/Attacker

```
Laptop with:
- 8GB RAM minimum (to run Kali VM or dual-boot) # I use 16gb
- Connection via switch (VLAN 30) or WiFi
```

### Physical Network

```
- Managed switch Gigabyte (any simple model)
- Cat5e/Cat6 Ethernet cables    # I use only cat6
- ISP Modem/Router
```

---

## Network Topology

### Physical Diagram

```
Internet
   │
   ├─── ISP Modem/Router
          │
          ├─── [enp5s0] Proxmox Server (Host)
          │       │
          │       ├─── vmbr0 (WAN) ──> pfSense WAN
          │       │
          │       ├─── vmbr1 (VLAN 10) ──> pfSense LAN + Proxmox Management
          │       │      │
          │       │      └─── [enx00e04c6801c8] ──> Dumb Switch
          │       │                                    │
          │       │                                    ├─── Physical Laptop
          │       │                                    └─── Access Point
          │       │
          │       ├─── vmbr2 (VLAN 20) ──> pfSense OPT1 + GOAD VMs
          │       │      └─── [VIRTUAL - NO PHYSICAL PORT]
          │       │
          │       ├─── vmbr3 (VLAN 30) ──> pfSense OPT2 + Kali VM
          │       │      └─── [VIRTUAL - NO PHYSICAL PORT]
          │       │
          │       └─── vmbr4 (VLAN 99) ──> pfSense OPT3 (future)
                         └─── [VIRTUAL - NO PHYSICAL PORT]
```

### VLAN Logical Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                         pfSense Firewall                       │
├──────────┬──────────┬──────────┬──────────┬────────────────────┤
│   WAN    │  VLAN 10 │  VLAN 20 │  VLAN 30 │     VLAN 99        │
│ Internet │    LAN   │   GOAD   │   Kali   │       WiFi         │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬───────────────┘
     │          │          │          │          │
     │     ┌────┴────┐     │     ┌────┴────┐     │ 
     │     │Proxmox  │     │     │ Kali VM │     │
     │     │ Switch  │     │     │192.168  │     │
     │     │Laptop   │     │     │ .30.50  │     │
     │     └─────────┘     │     └────┬────┘     │
     │                     │          │          │
     │              ┌──────┴──────┐   │Attack    │      
     │              │   GOAD Lab  │   │  ↓       │   
     │              │  ISOLATED!  │◄──┘          │    
     │              │             │              │
     │              │ DC01  DC02  │              │
     │              │ SRV01 SRV02 │              │
     │              │ WS01  WS02  │              │
     │              └─────────────┘              │
     │                                           │
     └───────────────────────────────── (Future: Wazuh, Sec Onion)
```

### Traffic Flows

#### Allowed

- **VLAN 10 → Internet** (browsing, updates)
- **VLAN 20 ↔ VLAN 20** (GOAD VMs among themselves)
- **VLAN 30 → VLAN 20** (Kali attacks GOAD)
- **VLAN 99 → Internet**

#### Blocked

- **VLAN 20 → Internet** (GOAD isolated without gateway)
- **VLAN 20 → VLAN 10** (GOAD doesn't access home network)
- **VLAN 30 → VLAN 10** (Kali doesn't access home network)
- **VLAN 10 → VLAN 20** (home network doesn't access GOAD)

---

## Installation Guides

### Implementation Order

1. **[Proxmox VE Installation](docs/01-proxmox-installation.md)**
2. **[Proxmox Network Configuration](docs/02-proxmox-network.md)**
3. **[pfSense Installation and Configuration](docs/03-pfsense-setup.md)**
4. **[VLAN Creation and Firewall Rules](docs/04-vlan-firewall-rules.md)**
5. **[GOAD Lab Deployment](docs/05-goad-deployment.md)**
6. **[Kali Linux Configuration](docs/06-kali-setup.md)**
7. **[Connectivity Tests and Validation](docs/07-testing-validation.md)**

### Additional Documents

- **[Complete Troubleshooting](docs/TROUBLESHOOTING.md)** ⭐ Issues encountered and solutions
- **[Useful Commands](docs/USEFUL-COMMANDS.md)** - Quick reference
- **[GOAD Pentesting Guide](docs/PENTESTING-GUIDE.md)** - How to attack the lab     # Coming son...
- **[FAQ](docs/FAQ.md)** - Frequently asked questions

---

## Use Cases

### 1. Active Directory Pentesting

```bash
# From Kali (192.168.40.50) attack GOAD (192.168.20.x)
nmap -sV -sC 192.168.20.0/24
crackmapexec smb 192.168.20.0/24 -u '' -p ''
bloodhound-python -d sevenkingdoms.local -u user -p pass -ns 192.168.20.10
```

### 2. Traffic Analysis

```bash
# Capture attack traffic on pfSense
tcpdump -i vtnet2 -w /tmp/goad-attack.pcap host 192.168.40.50
# Analyze in Wireshark later
```

### 3. Incident Response Simulation

```bash
# Wazuh/Security Onion monitoring VLAN 20
# Detect lateral movement, privilege escalation, etc.
```

---

## VM Specifications

### pfSense

```yaml
CPU: 2 cores
RAM: 2GB
Disk: 16GB
Network:
  - net0: vmbr1 (WAN)
  - net1: vmbr2 (VLAN 10 - LAN)
  - net2: vmbr0 (VLAN 20 - OPT1 GOAD)
  - net3: vmbr3 (VLAN 30 - OPT2 Kali)
  - net4: vmbr4 (VLAN 99 - OPT3 WiFi - and Quarentena,sub network of Access Point make for IoT and Guest)
```

### GOAD VMs (6 total)

```yaml
DC01 (Primary Domain Controller):
  OS: Windows Server 2019
  CPU: 2 cores
  RAM: 3GB
  Disk: 50GB
  IP: 192.168.20.10
  Roles: AD DS, DNS

DC02 (Backup Domain Controller):
  OS: Windows Server 2019
  CPU: 2 cores
  RAM: 3GB
  Disk: 50GB
  IP: 192.168.20.11
  Roles: AD DS, DNS

SRV01, SRV02 (Member Servers):
  OS: Windows Server 2019
  CPU: 2 cores
  RAM: 2GB
  Disk: 40GB
  IPs: 192.168.20.20, 192.168.20.21

WS01, WS02 (Workstations):
  OS: Windows 10
  CPU: 2 cores
  RAM: 2GB
  Disk: 40GB
  IPs: 192.168.20.30, 192.168.20.31
```

### Kali Linux - VM on laptop with Arch Linux

```yaml
CPU: 4 cores
RAM: 8GB
Disk: 450GB
Network: vmbr4 (VLAN 30)
IP: 192.168.30.50

**Performance Notes:**
This document compiles recommendations. I use Kali as a VM with a dedicated SSD, 
inside my personal computer with Arch Linux. If you use VM directly with QEMU 
instead of VirtualBox on Linux systems, you will certainly gain performance due 
to native KVM virtualization and reduced overhead.
```

---

## Future Roadmap

### Short Term

- [ ] Implement monitoring with Wazuh SIEM
- [ ] Add Security Onion for traffic analysis
- [ ] Document specific GOAD exploits
- [ ] Create automation scripts for deployment

### Medium Term

- [ ] Expand to 32GB RAM (more simultaneous VMs)
- [ ] Implement automated backup
- [ ] Create pre-exploitation snapshots
- [ ] Document complete red team operations

---

## Contributing

Contributions are welcome! If you found an issue or have an improvement:

1. Fork the repository
2. Create a branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add: new feature'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

---

## License

This project is under the MIT license. See the [LICENSE](LICENSE) file for more details.

---

## Acknowledgments

- **[Orange Cyberdefense](https://github.com/Orange-Cyberdefense/GOAD)** - GOAD creators
- **Proxmox Team** - Excellent virtualization platform
- **pfSense Team** - Robust open-source firewall
- **Brazilian Cybersecurity Community** - Support and inspiration

---

## Contact

- **GitHub:** [@lucasfelz](https://github.com/lucasfelz)
- **LinkedIn:** [Lucas Felz](https://linkedin.com/in/lucasfelz)
- **Email:** lucasfelz@gmail.com

---

## Disclaimer

This laboratory is intended **exclusively for educational purposes and information security research**. 

**NEVER use the techniques learned here on systems you do not have explicit authorization to test.**

The misuse of information contained in this project is the **sole responsibility of the user**.

---

<div align="center">

**[⬆ Back to top](#-cybersecurity-homelab-with-goad-lab)**

Made with ☕ and 🔐 | 2025

</div>
