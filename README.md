# Homelab de Cibersegurança com GOAD Lab

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Proxmox](https://img.shields.io/badge/Proxmox-VE%208-orange)](https://www.proxmox.com/)
[![pfSense](https://img.shields.io/badge/pfSense-2.7-blue)](https://www.pfsense.org/)
[![GOAD](https://img.shields.io/badge/GOAD-Active%20Directory%20Lab-red)](https://github.com/Orange-Cyberdefense/GOAD)

> **Homelab profissional para Cibersegurança, Pentesting e estudos de Active Directory com ambiente GOAD (Game of Active Directory) totalmente isolado.**

---

## Índice

- [Sobre o Projeto](#-sobre-o-projeto)
- [Arquitetura](#-arquitetura)
- [Requisitos de Hardware](#-requisitos-de-hardware)
- [Topologia de Rede](#-topologia-de-rede)
- [Guias de Instalação](#-guias-de-instalação)
- [Solução de Problemas](#-solução-de-problemas)
- [Referências](#-referências)

---

## Sobre o Projeto

Este projeto documenta a implementação completa de um homelab focado em cibersegurança e pentesting, com as seguintes características:

### Principais Recursos

- **Ambiente GOAD (Game of Active Directory)** - Lab com 6 VMs Windows vulneráveis
- **Isolamento completo de rede** - GOAD sem acesso à internet para máxima segurança
- **Rede ofensiva separada** - Kali Linux em VLAN dedicada
- **pfSense como firewall/roteador** - Controlador central para VLANs
- **Proxmox VE** - Virtualização de nível empresarial
- **Switch SF300-24** - Switch gerenciado 24 portas 10/100mbps + 4 portas Gigabit para tagging

### Objetivos de Aprendizado

- Pentesting em ambientes Active Directory
- Exploração de vulnerabilidades Windows (Kerberoasting, AS-REP Roasting, etc.)
- Segmentação de rede e firewalling
- Virtualização e gerenciamento de infraestrutura
- Análise de tráfego e resposta a incidentes

---

## Arquitetura

### Componentes Principais

| Componente | Descrição | Especificações |
|------------|-----------|----------------|
| **Proxmox VE 8.4** | Host de virtualização | Xeon E5-2630 v4, 16GB RAM |
| **pfSense** | Firewall/Roteador Virtual | 5 interfaces de rede, roteamento inter-VLAN |
| **GOAD Lab** | 6 VMs Windows vulneráveis | Domain Controllers, Servers, Workstations (Em progresso) |
| **Kali Linux** | VM para pentesting | Interface na VLAN 30 (ofensiva) |
| **Cisco SF300-24** | Switch gerenciado | 24x 10/100 Mbps + 4x Gigabit, suporte VLAN |

### VLANs Implementadas

| VLAN | Nome | Rede | Descrição | Internet |
|------|------|------|-----------|----------|
| **1** | WAN | DHCP | Internet via ISP | ✅ |
| **10** | Home LAN | 192.168.10.0/24 | Rede principal, gerenciamento Proxmox | ✅ |
| **20** | GOAD Lab | 192.168.20.0/24 | Ambiente Active Directory vulnerável (ISOLADO) | ❌ |
| **30** | Gerenciamento | 192.168.30.0/24 | Máquina de ataque, acesso limitado | ⚠️ Limitado |
| **99** | QUARENTENA (Huawei Access Point) | 192.168.99.0/24 | Rede WiFi isolada, acesso limitado | ⚠️ Limitado |

---

## Requisitos de Hardware

### Servidor Proxmox

```
CPU:     Intel Xeon E5-2630 v4 (10 cores, 20 threads)
RAM:     16GB DDR4 (mínimo) - Recomendado: 32GB+
Storage: SSD 256GB+ (para VMs GOAD) + HDD para backups
NICs:    2x Ethernet (1x onboard + 1x USB)
         - eth0 (enp5s0): WAN/Internet
         - eth1 (enx00e04c6801c8): LAN/Switch      # Uso adaptador USB para Ethernet com 2500mpbs
```

### Cliente/Atacante

```
Laptop com:
- 8GB RAM mínimo (para rodar VM Kali ou dual-boot) # Uso 16gb
- Conexão via switch (VLAN 30) ou WiFi
```

### Rede Física

```
- Switch gerenciado Gigabyte (qualquer modelo simples)
- Cabos Ethernet Cat5e/Cat6    # Uso apenas cat6
- Modem/Roteador ISP
```

---

## Topologia de Rede

### Diagrama Físico

```
Internet
   │
   ├─── Modem/Roteador ISP
          │
          ├─── [enp5s0] Servidor Proxmox (Host)
          │       │
          │       ├─── vmbr0 (WAN) ──> pfSense WAN
          │       │
          │       ├─── vmbr1 (VLAN 10) ──> pfSense LAN + Gerenciamento Proxmox
          │       │      │
          │       │      └─── [enx00e04c6801c8] ──> Switch Simples
          │       │                                    │
          │       │                                    ├─── Laptop Físico
          │       │                                    └─── Access Point
          │       │
          │       ├─── vmbr2 (VLAN 20) ──> pfSense OPT1 + VMs GOAD
          │       │      └─── [VIRTUAL - SEM PORTA FÍSICA]
          │       │
          │       ├─── vmbr3 (VLAN 30) ──> pfSense OPT2 + VM Kali
          │       │      └─── [VIRTUAL - SEM PORTA FÍSICA]
          │       │
          │       └─── vmbr4 (VLAN 99) ──> pfSense OPT3 (futuro)
                         └─── [VIRTUAL - SEM PORTA FÍSICA]
```

### Diagrama Lógico de VLANs

```
┌──────────────────────────────────────────────────────────────┐
│                         pfSense Firewall                     │
├──────────┬──────────┬──────────┬──────────┬────────────────────┤
│   WAN    │  VLAN 10 │  VLAN 20 │  VLAN 30 │     VLAN 99        │
│ Internet │    LAN   │   GOAD   │   Kali   │       WiFi         │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬───────────────┘
     │          │          │          │          │
     │     ┌────┴────┐     │     ┌────┴────┐     │ 
     │     │Proxmox  │     │     │ VM Kali │     │
     │     │ Switch  │     │     │192.168  │     │
     │     │Laptop   │     │     │ .30.50  │     │
     │     └─────────┘     │     └────┬────┘     │
     │                     │          │          │
     │              ┌──────┴──────┐   │Ataque    │      
     │              │   GOAD Lab  │   │  ↓       │   
     │              │  ISOLADO!   │◄──┘          │    
     │              │             │              │
     │              │ DC01  DC02  │              │
     │              │ SRV01 SRV02 │              │
     │              │ WS01  WS02  │              │
     │              └─────────────┘              │
     │                                           │
     └───────────────────────────────────── (Futuro: Wazuh, Sec Onion)
```

### Fluxos de Tráfego

#### Permitidos

- **VLAN 10 → Internet** (navegação, atualizações)
- **VLAN 20 ↔ VLAN 20** (VMs GOAD entre si)
- **VLAN 30 → VLAN 20** (Kali ataca GOAD)
- **VLAN 99 → Internet**

#### Bloqueados

- **VLAN 20 → Internet** (GOAD isolado sem gateway)
- **VLAN 20 → VLAN 10** (GOAD não acessa rede doméstica)
- **VLAN 30 → VLAN 10** (Kali não acessa rede doméstica)
- **VLAN 10 → VLAN 20** (rede doméstica não acessa GOAD)

---

## Guias de Instalação

### Ordem de Implementação

1. **[Instalação do Proxmox VE](docs/01-proxmox-installation.md)**
2. **[Configuração de Rede do Proxmox](docs/02-proxmox-network.md)**
3. **[Instalação e Configuração do pfSense](docs/03-pfsense-setup.md)**
4. **[Criação de VLANs e Regras de Firewall](docs/04-vlan-firewall-rules.md)**
5. **[Implantação do GOAD Lab](docs/05-goad-deployment.md)**
6. **[Configuração do Kali Linux](docs/06-kali-setup.md)**
7. **[Testes de Conectividade e Validação](docs/07-testing-validation.md)**

### Documentos Adicionais

- **[Solução Completa de Problemas](docs/TROUBLESHOOTING.md)** ⭐ Problemas encontrados e soluções
- **[Comandos Úteis](docs/USEFUL-COMMANDS.md)** - Referência rápida
- **[Guia de Pentesting GOAD](docs/PENTESTING-GUIDE.md)** - Como atacar o lab     # Em breve...
- **[FAQ](docs/FAQ.md)** - Perguntas frequentes

---

## Casos de Uso

### 1. Pentesting em Active Directory

```bash
# Do Kali (192.168.30.50) atacar GOAD (192.168.20.x)
nmap -sV -sC 192.168.20.0/24
crackmapexec smb 192.168.20.0/24 -u '' -p ''
bloodhound-python -d sevenkingdoms.local -u user -p pass -ns 192.168.20.10
```

### 2. Análise de Tráfego

```bash
# Capturar tráfego de ataque no pfSense
tcpdump -i vtnet2 -w /tmp/goad-attack.pcap host 192.168.30.50
# Analisar no Wireshark depois
```

### 3. Simulação de Resposta a Incidentes

```bash
# Wazuh/Security Onion monitorando VLAN 20
# Detectar movimento lateral, escalação de privilégios, etc.
```

---

## Especificações das VMs

### pfSense

```yaml
CPU: 2 cores
RAM: 2GB
Disco: 16GB
Rede:
  - net0: vmbr1 (WAN)
  - net1: vmbr2 (VLAN 10 - LAN)
  - net2: vmbr0 (VLAN 20 - OPT1 GOAD)
  - net3: vmbr3 (VLAN 30 - OPT2 Kali)
  - net4: vmbr4 (VLAN 99 - OPT3 WiFi - e Quarentena, sub-rede do Access Point para IoT e Convidados)
```

### VMs GOAD (6 no total)

```yaml
DC01 (Domain Controller Primário):
  OS: Windows Server 2019
  CPU: 2 cores
  RAM: 3GB
  Disco: 50GB
  IP: 192.168.20.10
  Funções: AD DS, DNS

DC02 (Domain Controller Backup):
  OS: Windows Server 2019
  CPU: 2 cores
  RAM: 3GB
  Disco: 50GB
  IP: 192.168.20.11
  Funções: AD DS, DNS

SRV01, SRV02 (Servidores Membros):
  OS: Windows Server 2019
  CPU: 2 cores
  RAM: 2GB
  Disco: 40GB
  IPs: 192.168.20.20, 192.168.20.21

WS01, WS02 (Estações de Trabalho):
  OS: Windows 10
  CPU: 2 cores
  RAM: 2GB
  Disco: 40GB
  IPs: 192.168.20.30, 192.168.20.31
```

### Kali Linux - VM no laptop com Arch Linux

```yaml
CPU: 4 cores
RAM: 8GB
Disco: 450GB
Rede: vmbr4 (VLAN 30)
IP: 192.168.30.50

**Notas sobre Performance:**
O presente documento reúne recomendações. Eu utilizo o Kali como VM com SSD dedicado, 
dentro do meu computador pessoal com Arch Linux. Se você utilizar VM direto com QEMU 
ao invés do VirtualBox nos sistemas Linux, você certamente terá ganho de performance 
devido à virtualização nativa KVM e menor overhead.
```

---

## Roadmap Futuro

### Curto Prazo

- [ ] Implementar monitoramento com Wazuh SIEM
- [ ] Adicionar Security Onion para análise de tráfego
- [ ] Documentar exploits específicos do GOAD
- [ ] Criar scripts de automação para implantação

### Médio Prazo

- [ ] Expandir para 32GB RAM (mais VMs simultâneas)
- [ ] Implementar backup automatizado
- [ ] Criar snapshots pré-exploração
- [ ] Documentar operações completas de red team

---

## Contribuindo

Contribuições são bem-vindas! Se você encontrou um problema ou tem uma melhoria:

1. Faça um Fork do repositório
2. Crie um branch (`git checkout -b feature/melhoria`)
3. Commit suas mudanças (`git commit -am 'Add: nova funcionalidade'`)
4. Push para o branch (`git push origin feature/melhoria`)
5. Abra um Pull Request

---

## Licença

Este projeto está sob a licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.

---

## Agradecimentos

- **[Orange Cyberdefense](https://github.com/Orange-Cyberdefense/GOAD)** - Criadores do GOAD
- **Proxmox Team** - Excelente plataforma de virtualização
- **pfSense Team** - Firewall open-source robusto
- **Comunidade Brasileira de Cibersegurança** - Suporte e inspiração

---

## Contato

- **GitHub:** [@lucasfelz](https://github.com/lucasfelz)
- **LinkedIn:** [Lucas Felz](https://linkedin.com/in/lucasfelz)
- **Email:** lucasfelz@gmail.com

---

## Aviso Legal

Este laboratório é destinado **exclusivamente para fins educacionais e pesquisa em segurança da informação**. 

**NUNCA use as técnicas aprendidas aqui em sistemas que você não tenha autorização explícita para testar.**

O uso indevido das informações contidas neste projeto é de **responsabilidade exclusiva do usuário**.

---

<div align="center">

**[⬆ Voltar ao topo](#-homelab-de-cibersegurança-com-goad-lab)**

Feito com ☕ e 🔐 | 2025

</div>
