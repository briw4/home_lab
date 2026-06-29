# Homelab Cybersecurity Project

This project is currently a work in progress.

## Overview

This repository documents the design and setup of a cybersecurity homelab using OPNsense, Windows, Kali Linux, Active Directory, and a SIEM (Wazuh).

The goal is to build a realistic environment for:
- Active Directory attack simulations
- Network security testing
- SIEM monitoring and detection
- Firewall and VLAN segmentation practice


## Current Status

✔ Phase 1: Network design and architecture (IN PROGRESS)  
✔ Phase 2: OPNsense setup (PLANNED)  
✔ Phase 3: Active Directory deployment  
✖ Phase 4: Attack simulations (Kali Linux)  
✖ Phase 5: Detection & logging (Wazuh / Suricata)


## Current Architecture

- 10.0.1.0/24 → Client Network (Windows, Kali)
- 10.0.2.0/24 → Server Network (AD, Wazuh)
- OPNsense → Firewall / router between networks


## Disclaimer

This environment is strictly for educational and ethical cybersecurity learning purposes.
