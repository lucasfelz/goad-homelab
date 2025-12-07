# Documentation Structure

Complete Arch Wiki-style documentation for GOAD Homelab project.

## Overview

Total documentation: **5,758 lines** across 10 files.

Documentation style: **Arch Wiki** - objective, concise, no fluff.

## File structure

```
goad-homelab/
├── README.md                          (Main documentation entry point)
├── LICENSE                            (MIT License)
└── docs/
    ├── 01-proxmox-installation.md     (Proxmox VE installation guide)
    ├── 02-proxmox-network.md          (Network bridge configuration)
    ├── 03-pfsense-setup.md            (pfSense firewall deployment)
    ├── 04-vlan-firewall-rules.md      (VLAN isolation and security)
    ├── 05-goad-deployment.md          (Active Directory lab setup)
    ├── 06-kali-setup.md               (Attack platform configuration)
    ├── 07-testing-validation.md       (Complete validation tests)
    ├── TROUBLESHOOTING.md             (Comprehensive problem-solving)
    └── USEFUL-COMMANDS.md             (Quick command reference)
```

## Document details

### README.md (374 lines)
Main entry point with complete architecture overview, component specifications, network topology, and installation roadmap.

**Key sections:**
- Architecture overview with component table
- VLAN configuration matrix
- Network topology diagrams
- Traffic flow rules
- VM specifications
- Usage examples

### Installation guides (Sequential order)

**01-proxmox-installation.md (183 lines)**
- ISO download and bootable USB creation
- Step-by-step installation process
- Post-installation configuration
- Repository setup
- IOMMU enablement

**02-proxmox-network.md (219 lines)**
- Dual-NIC configuration
- Bridge creation (vmbr0-4)
- Static IP assignment
- DNS configuration
- Network troubleshooting

**03-pfsense-setup.md (296 lines)**
- VM creation with 5 network interfaces
- pfSense installation wizard
- Interface assignment and configuration
- Web interface setup
- Initial security hardening

**04-vlan-firewall-rules.md (405 lines)**
- Complete firewall rule matrix
- VLAN isolation configuration
- NAT setup
- Port forwarding
- Rule verification and logging

**05-goad-deployment.md (504 lines)**
- Terraform automated deployment
- Ansible playbook execution
- Manual deployment alternative
- Active Directory configuration
- Domain join procedures

**06-kali-setup.md (457 lines)**
- Kali Linux installation
- Tool installation (BloodHound, CrackMapExec, Impacket)
- Network configuration on VLAN 40
- Attack workflow scripts
- BloodHound database setup

**07-testing-validation.md (492 lines)**
- Network connectivity tests
- VLAN isolation verification
- Active Directory health checks
- Performance benchmarks
- Automated test scripts

### Reference documents

**TROUBLESHOOTING.md (752 lines)**
Comprehensive problem-solving guide organized by component:
- Network issues (connectivity, DNS, routing)
- Proxmox issues (web interface, VM failures, storage)
- pfSense issues (interface down, firewall rules, NAT)
- GOAD issues (domain join, replication, Ansible)
- Kali issues (tools, BloodHound)
- Performance issues (CPU, memory, network)
- Emergency recovery procedures

**USEFUL-COMMANDS.md (282 lines)**
Quick reference for common operations:
- Proxmox VM management and snapshots
- pfSense firewall and network commands
- Active Directory PowerShell cmdlets
- Kali pentesting commands
- Network diagnostics
- System administration
- Backup procedures

## Documentation philosophy

Following Arch Wiki principles:

### ✅ What this documentation IS:

- **Objective** - Direct instructions without unnecessary commentary
- **Concise** - Minimum words needed to convey information
- **Structured** - Consistent format across all documents
- **Technical** - Assumes basic competence, explains when necessary
- **Practical** - Commands ready to copy-paste
- **Complete** - Covers installation through troubleshooting

### ❌ What this documentation is NOT:

- Tutorial-style with hand-holding
- Marketing material with excessive enthusiasm
- Opinion pieces about best practices
- Comparison with other solutions
- Philosophical discussions about technology choices

## Key features

**Troubleshooting format:**
```
Symptom: Clear description of problem
Diagnosis: Commands to identify issue
Common causes: List of likely reasons
Solution: Step-by-step fix
```

**Command format:**
```bash
# Comment explaining purpose
command with options

# Expected output or result
```

**Note/Warning format:**
```
**Note:** Additional context
**Warning:** Critical information
**Tip:** Optimization suggestion
```

## Usage

### For installation:

Follow documents in sequential order (01-07).

Each document ends with "Next steps" linking to next guide.

### For troubleshooting:

1. Check symptoms in TROUBLESHOOTING.md
2. Follow diagnosis commands
3. Apply solution
4. Verify fix

### For reference:

Use USEFUL-COMMANDS.md as quick lookup for common operations.

## Success criteria

Documentation is successful when:

- User can deploy complete homelab without external resources
- Every common issue has documented solution
- Commands are copy-pasteable and work correctly
- Troubleshooting leads to resolution
- No unnecessary words waste reader's time

## Statistics

| Metric | Value |
|--------|-------|
| Total lines | 5,758 |
| Total files | 10 |
| Installation guides | 7 |
| Reference docs | 2 |
| Troubleshooting scenarios | 30+ |
| Command examples | 200+ |
| Code blocks | 300+ |

## Maintenance

Documentation follows these principles:

1. **DRY** - No repeated information, link instead
2. **Update** - Keep commands current with software versions
3. **Test** - Verify all commands work before documenting
4. **Simplify** - Remove unnecessary complexity
5. **Organize** - Maintain consistent structure

## Contributing

When adding new content:

1. Follow Arch Wiki style
2. Use existing document structure
3. Add to appropriate section
4. Update this index
5. Test all commands
6. Keep formatting consistent

## See also

* [Arch Wiki Style Guide](https://wiki.archlinux.org/title/Help:Style)
* [Arch Wiki Writing Guidelines](https://wiki.archlinux.org/title/Help:Reading)
* [Main README](../README.md)

---

**Documentation created:** 2025-01-02  
**Style reference:** Arch Wiki  
**License:** MIT
