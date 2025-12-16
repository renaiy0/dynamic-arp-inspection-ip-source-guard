# Dynamic ARP Inspection (DAI) + IP Source Guard (IPSG) Security Lab

## Overview
This repository demonstrates the implementation of two critical Layer 2 security features on Cisco switches: Dynamic ARP Inspection (DAI) and IP Source Guard (IPSG). These features work together with DHCP Snooping to prevent ARP spoofing, IP spoofing, and man-in-the-middle attacks in switched networks.

![Network Topology](assets/dai-ipsg-topology.png)

## What is Dynamic ARP Inspection (DAI)?

Dynamic ARP Inspection is a security feature that validates ARP packets in a network. DAI intercepts, logs, and discards ARP packets with invalid IP-to-MAC address bindings, preventing:

- **ARP Spoofing/Poisoning** - Attacker sends fake ARP responses
- **Man-in-the-Middle Attacks** - Traffic interception via ARP poisoning
- **DoS Attacks** - ARP cache poisoning causing network disruption
- **Session Hijacking** - Stealing active sessions through ARP manipulation

## What is IP Source Guard (IPSG)?

IP Source Guard is a security feature that restricts IP traffic on untrusted Layer 2 ports by filtering traffic based on the DHCP Snooping binding database. IPSG prevents:

- **IP Address Spoofing** - Using unauthorized IP addresses
- **DHCP Bypass** - Manually configuring IP without DHCP
- **Network Scanning** - Using multiple IPs for reconnaissance
- **DoS Attacks** - Launching attacks from spoofed IPs

## How They Work Together

```
┌─────────────────────────────────────────────────────┐
│           DHCP SNOOPING (Foundation)                │
│     Builds binding database: MAC-IP-VLAN-Port       │
└────────────────┬────────────────────────────────────┘
                 │
        ┌────────┴─────────┐
        │                  │
        ▼                  ▼
┌──────────────┐    ┌─────────────────┐
│     DAI      │    │  IP Source Guard│
│  Validates   │    │   Validates     │
│  ARP packets │    │   IP packets    │
└──────────────┘    └─────────────────┘
```

Both features rely on the **DHCP Snooping binding database** to validate traffic.

## Prerequisites

Before configuring DAI and IPSG, you MUST have:

1. **DHCP Snooping enabled and configured**
```cisco
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 1
```

2. **Trusted and untrusted ports properly configured**
```cisco
Switch(config)# interface fa0/1
Switch(config-if)# ip dhcp snooping trust
```

Without DHCP Snooping, DAI and IPSG will not function properly.

## Configuration Overview

![Configuration Commands](assets/config-commands.png)

### Trust State Configuration

The image shows the trust state for all interfaces:

```
Interface    Trust State    Rate(pps)    Burst Interval
---------    -----------    ---------    --------------
Fa0/1        Trusted        15           1
Fa0/2        Untrusted      15           1
Fa0/3        Untrusted      15           1
Fa0/4        Untrusted      15           1
Fa0/5        Trusted        15           1
Fa0/6-20     Untrusted      15           1
```

**Key Points:**
- **Fa0/1 & Fa0/5**: Trusted (uplinks/servers)
- **Fa0/2-4, Fa0/6-20**: Untrusted (client ports)
- **Rate limiting**: 15 ARP packets per second
- **Burst interval**: 1 second

## Step-by-Step Configuration

### Step 1: Enable DHCP Snooping (Required)

```cisco
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 1
Switch(config)# no ip dhcp snooping information option

! Configure trusted ports
Switch(config)# interface range fa0/1, fa0/5
Switch(config-if-range)# ip dhcp snooping trust
```

![DHCP Snooping Setup](assets/dhcp-snooping-base.png)

### Step 2: Enable Dynamic ARP Inspection

```cisco
! Enable DAI on VLAN 1
Switch(config)# ip arp inspection vlan 1

! Configure trusted ports for DAI
Switch(config)# interface range fa0/1, fa0/5
Switch(config-if-range)# ip arp inspection trust

! Configure rate limiting (optional but recommended)
Switch(config)# interface range fa0/2-4, fa0/6-20
Switch(config-if-range)# ip arp inspection limit rate 15
```

![DAI Configuration](assets/dai-config.png)

### Step 3: Enable IP Source Guard

```cisco
! Enable IP Source Guard on untrusted ports
Switch(config)# interface range fa0/1, fa0/5
Switch(config-if-range)# ip verify source

! For stricter security, verify both IP and MAC
Switch(config)# interface range fa0/2-4, fa0/6-20
Switch(config-if-range)# ip verify source port-security
```

![IPSG Configuration](assets/ipsg-config.png)

### Step 4: Verify Configuration

```cisco
! Verify DAI configuration
Switch# show ip arp inspection

! Verify DAI statistics
Switch# show ip arp inspection statistics

! Verify DAI interfaces
Switch# show ip arp inspection interfaces

! Verify IP Source Guard
Switch# show ip verify source

! Verify DHCP Snooping binding (used by DAI and IPSG)
Switch# show ip dhcp snooping binding
```

![Verification Output](assets/verification.png)

## Complete Configuration Example

### Full Switch Configuration

```cisco
Switch> enable
Switch# configure terminal

!--- DHCP Snooping Configuration (Foundation) ---
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 1
Switch(config)# no ip dhcp snooping information option

!--- Configure Trusted Ports ---
Switch(config)# interface fa0/1
Switch(config-if)# description UPLINK-OR-DHCP-SERVER
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# ip arp inspection trust
Switch(config-if)# exit

Switch(config)# interface fa0/5
Switch(config-if)# description TRUSTED-UPLINK
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# ip arp inspection trust
Switch(config-if)# exit

!--- Dynamic ARP Inspection Configuration ---
Switch(config)# ip arp inspection vlan 1

!--- Configure Untrusted Ports with DAI Rate Limiting ---
Switch(config)# interface range fa0/2-4, fa0/6-20
Switch(config-if-range)# description CLIENT-PORTS
Switch(config-if-range)# ip arp inspection limit rate 15
Switch(config-if-range)# exit

!--- IP Source Guard Configuration ---
Switch(config)# interface range fa0/1, fa0/5
Switch(config-if-range)# ip verify source
Switch(config-if-range)# exit

!--- Save Configuration ---
Switch(config)# exit
Switch# write memory
```

## Attack Scenarios Prevented

### 1. ARP Spoofing Attack (Prevented by DAI)

**Without DAI:**
```
Attacker → Fake ARP: "I am 192.168.1.1 (Gateway), my MAC is XX:XX:XX"
Victim   → Updates ARP cache with attacker's MAC
Traffic  → Goes through attacker instead of real gateway
Result   → Man-in-the-Middle attack successful ❌
```

**With DAI:**
```
Attacker → Fake ARP: "I am 192.168.1.1, my MAC is XX:XX:XX"
Switch   → Checks DHCP Snooping binding database
Switch   → Invalid binding detected (MAC doesn't match)
Switch   → Drops ARP packet
Result   → Attack prevented ✅
```

### 2. IP Spoofing Attack (Prevented by IPSG)

**Without IPSG:**
```
Attacker → Manually configures IP 192.168.1.100
Attacker → Sends packets with spoofed source IP
Network  → Accepts packets (no validation)
Result   → IP spoofing successful ❌
```

**With IPSG:**
```
Attacker → Manually configures IP 192.168.1.100
Attacker → Sends packets with spoofed source IP
Switch   → Checks DHCP Snooping binding database
Switch   → IP 192.168.1.100 not assigned to this port
Switch   → Drops packets
Result   → Attack prevented ✅
```

### 3. Man-in-the-Middle Attack

**Attack Chain:**
1. Attacker poisons ARP cache (blocked by DAI)
2. Attacker uses spoofed IP (blocked by IPSG)
3. Combined defense: Both attacks prevented

## Verification Commands

### Check DAI Status

```cisco
Switch# show ip arp inspection

Source Mac Validation      : Disabled
Destination Mac Validation : Disabled
IP Address Validation      : Disabled

Vlan     Configuration    Operation   ACL Match          Static ACL
----     -------------    ---------   ---------          ----------
   1     Enabled          Active

Vlan     ACL Logging      DHCP Logging      Probe Logging
----     -----------      ------------      -------------
   1     Deny             Deny              Off
```

### Check DAI Interfaces

```cisco
Switch# show ip arp inspection interfaces

Interface        Trust State     Rate (pps)    Burst Interval
--------------   -----------     ----------    --------------
Fa0/1            Trusted         15            1
Fa0/2            Untrusted       15            1
Fa0/5            Trusted         15            1
```

![DAI Interfaces](assets/dai-interfaces.png)

### Check DAI Statistics

```cisco
Switch# show ip arp inspection statistics

Vlan      Forwarded        Dropped     DHCP Drops      ACL Drops
----      ---------        -------     ----------      ---------
   1           245             12            12              0

Vlan   DHCP Permits    ACL Permits  Probe Permits   Source MAC Failures
----   ------------    -----------  -------------   -------------------
   1           233              0            0                  0

Vlan   Dest MAC Failures   IP Validation Failures   Invalid Protocol Data
----   -----------------   ----------------------   ---------------------
   1                0                      0                        0
```

### Check IP Source Guard

```cisco
Switch# show ip verify source

Interface  Filter-type  Filter-mode  IP-address       Mac-address        Vlan
---------  -----------  -----------  ---------------  -----------------  ----
Fa0/2      ip           active       192.168.1.10     permit-all         1
Fa0/3      ip           active       192.168.1.11     permit-all         1
Fa0/4      ip           active       deny-all         permit-all         1
```

![IP Source Guard Status](assets/ipsg-status.png)

### Check DHCP Snooping Binding

```cisco
Switch# show ip dhcp snooping binding

MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
00:1A:2B:3C:4D:5E   192.168.1.10     86395       dhcp-snooping   1    FastEthernet0/2
00:1A:2B:3C:4D:5F   192.168.1.11     86398       dhcp-snooping   1    FastEthernet0/3
```

## Advanced DAI Features

### 1. ARP Packet Validation

```cisco
! Validate source MAC address
Switch(config)# ip arp inspection validate src-mac

! Validate destination MAC address
Switch(config)# ip arp inspection validate dst-mac

! Validate IP addresses
Switch(config)# ip arp inspection validate ip

! Enable all validations
Switch(config)# ip arp inspection validate src-mac dst-mac ip
```

### 2. DAI with ACLs (Static Entries)

```cisco
! Create ARP ACL for static entries
Switch(config)# arp access-list STATIC-HOSTS
Switch(config-arp-nacl)# permit ip host 192.168.1.1 mac host 0011.2233.4455
Switch(config-arp-nacl)# exit

! Apply ACL to VLAN
Switch(config)# ip arp inspection filter STATIC-HOSTS vlan 1
```

### 3. DAI Logging

```cisco
! Configure DAI logging
Switch(config)# ip arp inspection log-buffer entries 128
Switch(config)# ip arp inspection log-buffer logs 64 interval 5

! View DAI log
Switch# show ip arp inspection log
```

## Advanced IPSG Features

### 1. IP Source Guard with MAC Filtering

```cisco
! First, enable port security
Switch(config)# interface fa0/2
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 2

! Then enable IPSG with MAC verification
Switch(config-if)# ip verify source port-security
```

### 2. Static IP Source Binding

```cisco
! For devices with static IPs (not using DHCP)
Switch(config)# ip source binding 0011.2233.4455 vlan 1 192.168.1.100 interface fa0/10

! Verify static bindings
Switch# show ip source binding
```

## Troubleshooting Guide

### Issue 1: Legitimate ARP Packets Dropped

**Symptoms:**
- Connectivity issues
- DAI statistics show drops
- "DHCP Drops" incrementing

**Solutions:**
```cisco
! Check if DHCP Snooping is enabled
Switch# show ip dhcp snooping

! Verify port trust state
Switch# show ip arp inspection interfaces

! Check binding database
Switch# show ip dhcp snooping binding

! If legitimate server, make port trusted
Switch(config)# interface fa0/1
Switch(config-if)# ip arp inspection trust
```

### Issue 2: IP Source Guard Blocking Traffic

**Symptoms:**
- New devices cannot communicate
- "deny-all" in IPSG status
- No DHCP binding for device

**Solutions:**
```cisco
! Check IPSG status
Switch# show ip verify source

! Verify DHCP Snooping binding
Switch# show ip dhcp snooping binding

! For static IP devices, add static binding
Switch(config)# ip source binding [MAC] vlan [VLAN] [IP] interface [INT]

! Or disable IPSG on specific port (not recommended)
Switch(config)# interface fa0/x
Switch(config-if)# no ip verify source
```

### Issue 3: High ARP Rate Limit Drops

**Symptoms:**
- Legitimate traffic affected
- DAI statistics show rate limit drops

**Solutions:**
```cisco
! Increase rate limit on trusted ports
Switch(config)# interface fa0/1
Switch(config-if)# ip arp inspection limit rate none

! Or increase rate limit value
Switch(config-if)# ip arp inspection limit rate 50
```

### Issue 4: DHCP Snooping Database Issues

**Symptoms:**
- Bindings disappearing after reload
- DAI/IPSG not working after reboot

**Solutions:**
```cisco
! Configure database storage location
Switch(config)# ip dhcp snooping database flash:/dhcp-snooping.db
Switch(config)# ip dhcp snooping database timeout 60

! Verify database
Switch# show ip dhcp snooping database
```

## Security Best Practices

### ✅ Configuration Checklist

- [x] Enable DHCP Snooping first (foundation)
- [x] Configure trusted ports for uplinks/servers
- [x] Enable DAI on all VLANs with DHCP
- [x] Configure DAI rate limiting on untrusted ports
- [x] Enable IPSG on all access ports
- [x] Enable ARP validation (src-mac, dst-mac, ip)
- [x] Configure static bindings for non-DHCP devices
- [x] Enable logging and monitoring
- [x] Document all trusted ports
- [x] Test thoroughly before production

### ✅ Deployment Strategy

1. **Phase 1**: Enable DHCP Snooping
2. **Phase 2**: Wait 24-48 hours, verify bindings
3. **Phase 3**: Enable DAI on test VLAN
4. **Phase 4**: Monitor for false positives
5. **Phase 5**: Enable IPSG on test ports
6. **Phase 6**: Gradual rollout to production

### ❌ Common Mistakes to Avoid

- Starting with DAI/IPSG without DHCP Snooping
- Forgetting to trust uplink ports
- Setting rate limits too low
- Not accounting for static IP devices
- Not saving binding database
- Not monitoring DAI/IPSG statistics

## Performance Considerations

### Rate Limiting Guidelines

| Port Type | Recommended Rate | Burst Interval |
|-----------|------------------|----------------|
| Access (Client) | 15-30 pps | 1 second |
| Server | 50-100 pps | 1 second |
| Uplink/Trunk | unlimited | N/A |
| Wireless AP | 30-50 pps | 1 second |

### CPU Impact

- DAI and IPSG process in hardware (TCAM) on modern switches
- Minimal CPU impact on Catalyst 2960 and newer
- Older switches may experience higher CPU usage

## Integration with Other Security Features

### Complete Layer 2 Security Stack

```cisco
!--- DHCP Snooping (Layer 1) ---
ip dhcp snooping
ip dhcp snooping vlan 1

!--- Port Security (Layer 2) ---
interface range fa0/2-24
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict

!--- Dynamic ARP Inspection (Layer 3) ---
ip arp inspection vlan 1
ip arp inspection validate src-mac dst-mac ip

!--- IP Source Guard (Layer 4) ---
interface range fa0/2-24
 ip verify source port-security

!--- Storm Control (Layer 5) ---
interface range fa0/2-24
 storm-control broadcast level 50
 storm-control multicast level 50
```

## Repository Structure

```
.
├── assets/
│   ├── dai-ipsg-topology.png
│   ├── config-commands.png
│   ├── dai-config.png
│   ├── ipsg-config.png
│   ├── dai-interfaces.png
│   ├── ipsg-status.png
│   ├── verification.png
│   └── dhcp-snooping-base.png
├── configs/
│   ├── complete-dai-ipsg-config.txt
│   ├── dhcp-snooping-foundation.txt
│   ├── dai-only-config.txt
│   └── ipsg-only-config.txt
├── documentation/
│   ├── dai-deployment-guide.pdf
│   ├── ipsg-best-practices.pdf
│   └── troubleshooting-guide.pdf
└── README.md
```

## Lab Testing Scenarios

### Test 1: Verify DAI Blocks Fake ARP

```
1. Configure attacker PC with manual IP
2. Use ARP spoofing tool
3. Verify DAI drops fake ARP packets
4. Check statistics: show ip arp inspection statistics
```

### Test 2: Verify IPSG Blocks IP Spoofing

```
1. Configure PC with manual IP (not via DHCP)
2. Attempt to send traffic
3. Verify IPSG blocks packets
4. Check status: show ip verify source
```

### Test 3: Verify Legitimate Traffic Passes

```
1. Client gets IP via DHCP
2. Verify binding: show ip dhcp snooping binding
3. Test connectivity (ping, web browsing)
4. Verify no DAI/IPSG drops
```

## Learning Objectives

This lab demonstrates:

1. Understanding ARP spoofing and IP spoofing attacks
2. Implementing Dynamic ARP Inspection (DAI)
3. Configuring IP Source Guard (IPSG)
4. Integration with DHCP Snooping
5. Trusted vs untrusted port configuration
6. Rate limiting and validation techniques
7. Troubleshooting Layer 2 security features
8. Building defense-in-depth security architecture

## Key Concepts

### DHCP Snooping Binding Database

The foundation for both DAI and IPSG:

```
MAC Address        IP Address      VLAN    Port
-------------      -----------     ----    ----
AA:BB:CC:DD:EE:FF  192.168.1.10    1       Fa0/2
11:22:33:44:55:66  192.168.1.11    1       Fa0/3
```

- **DAI** uses this to validate ARP packets
- **IPSG** uses this to validate IP packets
- Both check: "Is this IP/MAC combination valid for this port?"

## Additional Resources

- Cisco Dynamic ARP Inspection Configuration Guide
- Cisco IP Source Guard Configuration Guide
- Layer 2 Attack Mitigation Best Practices
- DHCP Snooping Deployment Guide

## Lab Environment

- **Platform**: Cisco Packet Tracer / GNS3
- **Switch**: Cisco Catalyst 2960/2970 series
- **Features**: DHCP Snooping, DAI, IPSG
- **Topology**: Multi-client with trusted uplinks

## Getting Started

1. Review DHCP Snooping configuration
2. Enable DAI on VLANs
3. Configure trusted ports for DAI
4. Enable IP Source Guard on access ports
5. Test with legitimate and malicious traffic
6. Monitor statistics and logs
7. Document trust configuration

## License

Educational/Training purposes

---

*This lab demonstrates enterprise-grade Layer 2 security implementation using Dynamic ARP Inspection and IP Source Guard to prevent spoofing attacks.*
