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