# VLAN and Firewall Rules

Firewall configuration for network segmentation and traffic control between VLANs.

## Prerequisites

* pfSense installed with all interfaces configured
* Understanding of firewall rule logic
* Access to pfSense web interface

## Rule philosophy

pfSense uses **default deny** policy. Traffic is blocked unless explicitly allowed by a rule.

**Rule evaluation:**

* Rules are processed top-to-bottom
* First matching rule wins
* Implicit deny at end of ruleset

## VLAN isolation matrix

Traffic flow requirements:

| Source | Destination | Action | Purpose |
|--------|-------------|--------|---------|
| VLAN 10 | Internet | Allow | Management access |
| VLAN 10 | VLAN 20 | Block | Prevent home network from accessing vulnerable lab |
| VLAN 10 | VLAN 30 | Block | Prevent home network from accessing attack platform |
| VLAN 20 | VLAN 20 | Allow | GOAD VMs communicate internally |
| VLAN 20 | Internet | Block | Complete isolation |
| VLAN 20 | All others | Block | Lab containment |
| VLAN 30 | VLAN 20 | Allow | Kali attacks GOAD |
| VLAN 30 | Internet | Allow (80/443) | Tool updates only |
| VLAN 30 | VLAN 10 | Block | Attack platform isolated from home network |
| VLAN 99 | Internet | Allow | WiFi/IoT limited access |

## Configure LAN rules (VLAN 10)

Navigate to:

```
Firewall > Rules > LAN
```

### Default rules

pfSense creates default LAN rules. Modify as needed.

**Rule 1: Allow LAN to any**

```
Action: Pass
Interface: LAN
Protocol: Any
Source: LAN net
Destination: Any
Description: Allow LAN to internet and other VLANs
```

**Note:** This is permissive. Consider restricting if security requires.

### Block access to vulnerable VLANs

Add rules above default allow rule:

**Block LAN to GOAD:**

```
Action: Block
Interface: LAN
Protocol: Any
Source: LAN net
Destination: VLAN20_GOAD net (192.168.20.0/24)
Description: Block home network from accessing GOAD lab
```

**Block LAN to Kali:**

```
Action: Block
Interface: LAN
Protocol: Any
Source: LAN net
Destination: VLAN30_KALI net (192.168.30.0/24)
Description: Block home network from accessing attack platform
```

Click Save and Apply Changes.

## Configure GOAD rules (VLAN 20)

Navigate to:

```
Firewall > Rules > VLAN20_GOAD
```

### Allow internal communication

**Rule 1: Allow GOAD to GOAD**

```
Action: Pass
Interface: VLAN20_GOAD
Protocol: Any
Source: VLAN20_GOAD net
Destination: VLAN20_GOAD net
Description: Allow internal GOAD communication
```

### Block all outbound traffic

**Rule 2: Block GOAD to any**

```
Action: Block
Interface: VLAN20_GOAD
Protocol: Any
Source: VLAN20_GOAD net
Destination: Any
Description: Block GOAD from leaving isolated network
Log: Check "Log packets matched by this rule"
```

**Important:** Place this rule after internal allow rule but before any default rules.

Final VLAN20_GOAD ruleset order:

```
1. Pass: GOAD net → GOAD net
2. Block: GOAD net → Any (logged)
```

## Configure Kali rules (VLAN 30)

Navigate to:

```
Firewall > Rules > VLAN30_KALI
```

### Allow Kali to GOAD

**Rule 1: Allow Kali to GOAD**

```
Action: Pass
Interface: VLAN30_KALI
Protocol: Any
Source: VLAN30_KALI net
Destination: VLAN20_GOAD net (192.168.20.0/24)
Description: Allow Kali to attack GOAD lab
```

### Allow limited internet (updates only)

**Rule 2: Allow HTTP/HTTPS**

```
Action: Pass
Interface: VLAN30_KALI
Protocol: TCP
Source: VLAN30_KALI net
Destination: Any
Destination port: HTTP (80)
Description: Allow Kali HTTP for updates
```

**Rule 3: Allow HTTPS**

```
Action: Pass
Interface: VLAN30_KALI
Protocol: TCP
Source: VLAN30_KALI net
Destination: Any
Destination port: HTTPS (443)
Description: Allow Kali HTTPS for updates
```

### Allow DNS

**Rule 4: Allow DNS**

```
Action: Pass
Interface: VLAN30_KALI
Protocol: UDP
Source: VLAN30_KALI net
Destination: Any
Destination port: DNS (53)
Description: Allow DNS resolution
```

### Block all other outbound

**Rule 5: Block Kali to LAN**

```
Action: Block
Interface: VLAN30_KALI
Protocol: Any
Source: VLAN30_KALI net
Destination: LAN net
Description: Block Kali from accessing home network
Log: Check
```

**Rule 6: Block remaining traffic**

```
Action: Block
Interface: VLAN30_KALI
Protocol: Any
Source: VLAN30_KALI net
Destination: Any
Description: Block all other Kali traffic
Log: Check
```

Final VLAN30_KALI ruleset order:

```
1. Pass: Kali → GOAD
2. Pass: Kali → Any (port 80)
3. Pass: Kali → Any (port 443)
4. Pass: Kali → Any (port 53 UDP)
5. Block: Kali → LAN (logged)
6. Block: Kali → Any (logged)
```

## Configure WiFi/Quarentena rules (VLAN 99)

Navigate to:

```
Firewall > Rules > VLAN99_WIFI
```

### Allow outbound traffic with limitations

**Rule 1: Allow WiFi to internet**

```
Action: Pass
Interface: VLAN99_WIFI
Protocol: Any
Source: VLAN99_WIFI net
Destination: Any
Description: Allow WiFi/IoT internet access
```

### Block access to internal networks

**Rule 2: Block WiFi to LAN**

```
Action: Block
Interface: VLAN99_WIFI
Protocol: Any
Source: VLAN99_WIFI net
Destination: LAN net
Description: Block WiFi from accessing home network
Log: Check
```

**Rule 3: Block WiFi to GOAD**

```
Action: Block
Interface: VLAN99_WIFI
Protocol: Any
Source: VLAN99_WIFI net
Destination: VLAN20_GOAD net
Description: Block WiFi from accessing GOAD lab
Log: Check
```

**Rule 4: Block WiFi to Kali**

```
Action: Block
Interface: VLAN99_WIFI
Protocol: Any
Source: VLAN99_WIFI net
Destination: VLAN30_KALI net
Description: Block WiFi from accessing attack platform
Log: Check
```

## NAT configuration

### Outbound NAT

Navigate to:

```
Firewall > NAT > Outbound
```

Verify mode:

```
Mode: Automatic outbound NAT
```

This creates automatic NAT for all interfaces with internet gateway.

**Custom NAT (if needed):**

Switch to Hybrid or Manual mode and add:

```
Interface: WAN
Source: VLAN10 net (192.168.10.0/24)
Destination: Any
NAT Address: WAN address
Description: NAT for VLAN 10
```

Repeat for VLAN 30 and VLAN 99 (VLAN 20 should NOT have NAT).

### Port forwarding (optional)

Navigate to:

```
Firewall > NAT > Port Forward
```

Example: Forward SSH to Kali VM:

```
Interface: WAN
Protocol: TCP
Destination: WAN address
Destination port: 2222
Redirect target IP: 192.168.30.50
Redirect target port: 22
Description: SSH to Kali
```

**Warning:** Exposing attack platform to internet is not recommended.

## Aliases for easier management

Navigate to:

```
Firewall > Aliases
```

### Create network aliases

**GOAD_Network:**

```
Name: GOAD_Network
Type: Network(s)
Network: 192.168.20.0/24
Description: GOAD Lab network
```

**Kali_Network:**

```
Name: Kali_Network
Type: Network(s)
Network: 192.168.30.0/24
Description: Kali attack platform network
```

**Home_Network:**

```
Name: Home_Network
Type: Network(s)
Network: 192.168.10.0/24
Description: Home LAN network
```

**WiFi_Network:**

```
Name: WiFi_Network
Type: Network(s)
Network: 192.168.99.0/24
Description: WiFi/Quarentena network
```

### Create port aliases

**Web_Ports:**

```
Name: Web_Ports
Type: Port(s)
Ports: 80 443
Description: HTTP and HTTPS ports
```

**Common_Ports:**

```
Name: Common_Ports
Type: Port(s)
Ports: 22 80 443 3389
Description: SSH, HTTP, HTTPS, RDP
```

Use aliases in rules for easier updates.

## Verification

### Test allowed traffic

**From VLAN 10 (home network):**

```bash
# Should succeed
ping 8.8.8.8
curl https://google.com

# Should fail (blocked)
ping 192.168.20.10
nc -zv 192.168.30.50 22
```

**From Kali VM (VLAN 30):**

```bash
# Should succeed
ping 192.168.20.10
nmap 192.168.20.0/24
curl https://google.com

# Should fail (blocked)
ping 192.168.10.1
ssh user@192.168.10.x
```

**From GOAD VMs (VLAN 20):**

```powershell
# Should succeed
ping 192.168.20.10

# Should fail (blocked)
ping 8.8.8.8
ping 192.168.10.1
```

### Monitor firewall logs

Navigate to:

```
Status > System Logs > Firewall
```

Look for blocked traffic to verify rules are working:

```
Block: GOAD attempting to reach 8.8.8.8
Block: Kali attempting to reach 192.168.10.x
```

**Tip:** Enable logging on block rules to see violation attempts.

### Check states table

Navigate to:

```
Diagnostics > States
```

Shows active connections through firewall. Useful for debugging.

## Advanced configuration

### Rate limiting (optional)

Prevent DoS from attack platform:

```
Firewall > Rules > VLAN30_KALI
Advanced Options:
  Max states: 1000
  Max source nodes: 5
  Max connections per source: 100
```

### Traffic shaping (optional)

Limit bandwidth for specific VLANs:

```
Firewall > Traffic Shaper
```

Example: Limit Kali to 10 Mbps.

### Geo-blocking (optional)

Block traffic from specific countries (requires pfBlockerNG package).

## Next steps

Proceed to [GOAD Deployment](05-goad-deployment.md) to set up vulnerable Active Directory environment.

## Troubleshooting

### Symptom: Traffic allowed that should be blocked

**Cause:** Rules in wrong order or floating rules overriding.

**Solution:**

Check rule order. Remember: first match wins.

```
Firewall > Rules > <Interface>
```

Drag rules to correct order (specific rules above general rules).

Check for floating rules:

```
Firewall > Rules > Floating
```

Disable or reorder floating rules.

### Symptom: All traffic blocked including allowed

**Cause:** Overly restrictive rule or missing allow rule.

**Solution:**

Temporarily add allow-all rule at top:

```
Action: Pass
Protocol: Any
Source: Any
Destination: Any
Description: TEMPORARY DEBUG RULE
```

Test connectivity. If works, issue is with rules below. Review and fix.

**Important:** Remove debug rule after troubleshooting.

### Symptom: NAT not working

**Cause:** Outbound NAT not configured for VLAN.

**Solution:**

Verify outbound NAT:

```
Firewall > NAT > Outbound
Mode: Automatic or Manual with rules for each VLAN
```

Check NAT rules cover source networks needing internet.

### Symptom: Cannot access pfSense from certain VLANs

**Cause:** Anti-lockout rule only applies to LAN by default.

**Solution:**

Enable access from other interfaces:

```
System > Advanced > Admin Access
Enable: Allow access from all interfaces (not recommended)
```

Or add firewall rule:

```
Interface: <VLAN_INTERFACE>
Action: Pass
Protocol: TCP
Destination: This firewall
Destination port: 443
Description: Allow web access to pfSense
```

## See also

* [pfSense Firewall Rules](https://docs.netgate.com/pfsense/en/latest/firewall/index.html)
* [pfSense NAT Configuration](https://docs.netgate.com/pfsense/en/latest/nat/index.html)
* [Previous: pfSense Setup](03-pfsense-setup.md)
* [Next: GOAD Deployment](05-goad-deployment.md)
