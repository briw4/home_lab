# IP Address Plan

## Overview

This document describes the IP addressing scheme used in the home lab.

**Design goals:**

- Separate client machines from servers
- Improve security through network segmentation
- Centralize authentication and DNS via Active Directory
- Enable security monitoring with Wazuh SIEM
- Control and inspect inter-network traffic with OPNsense
---

## Network

| Network        | Subnet         | Gateway     | Purpose                        |
|----------------|----------------|-------------|--------------------------------|
| Client Network | 10.0.1.0/24    | 10.0.1.1    | User and security testing VM  |
| Server Network | 10.0.2.0/24    | 10.0.2.1    | Infrastructure servers         |

---
## Firewall (OPNsense)

OPNsense acts as the central router/firewall between both networks and the internet.

| Interface      | IP Address   | Purpose                          |
|----------------|-------------|----------------------------------|
| WAN            | ISP/DHCP    | Internet access                  |
| LAN (Client)   | 10.0.1.1    | Gateway for client network       |
| OPT1 (Server)  | 10.0.2.1    | Gateway for server network       |

---
## Client Network — 10.0.1.0/24

**Subnet:** `10.0.1.0/24`  
**Gateway:** `10.0.1.1` (OPNsense)

| Device       | IP           | OS            | Role                        |
|--------------|-------------|---------------|-----------------------------|
| Windows 10   | 10.0.1.10    | Windows 10    | Client workstation          |
| Kali Linux   | 10.0.1.20    | Kali Linux     | Attacker / testing machine  |

**Notes:**
- Windows 10 and Kali Linux are on the same subnet for controlled attack and defense scenarios
- OPNsense monitors traffic via firewall rules and Suricata IDS on this interface

---
## Server Network — 10.0.2.0/24

**Subnet:** `10.0.2.0/24`  
**Gateway:** `10.0.2.1` (OPNsense)

| Device              | IP           | OS             | Role                              |
|---------------------|-------------|----------------|-----------------------------------|
| Active Directory DC | 10.0.2.10    | Windows Server | Active Directory + DNS server     |
| Wazuh Server        | 10.0.2.20    | Linux          | SIEM, log collection & alerting  |

**Notes:**
- The Domain Controller provides DNS for all machines in the lab
- All domain-joined machines use `10.0.2.10` as primary DNS
- Wazuh collects and correlates security events from both networks