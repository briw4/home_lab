# OPNsense Setup: Firewall & VLAN Routing

 **Role:** OPNsense acts as the router and firewall between the two networks.  
 **VM network adapters:** 3 interfaces: WAN, LAN/VLAN1, OPT1/VLAN2


## Interface Summary

| Interface | Adapter | IP            | Role                        |
|-----------|---------|---------------|-----------------------------|
| WAN       | em0     | DHCP          | Internet / host network     |
| LAN       | em1     | 10.0.1.1/24   | Client network (VLAN1)      |
| OPT1      | em2     | 10.0.2.1/24   | Server network (VLAN2)      |

## Step 1 - Interface Assignment

When OPNsense boots for the first time it opens a console menu. Choose option `1` to assign interfaces.

```
Do you want to configure VLANs? → No
Enter WAN interface name      → em0
Enter LAN interface name      → em1
Enter OPT1 interface name     → em2
Do you want to proceed?       → y
```

## Step 2 - Set IP Addresses

From the console menu choose option `2` to set interface IPs.

### LAN (em1) - Client network

```
Available interfaces:
1 - WAN
2 - LAN
3 - OPT1

Select interface → 2

Enter new LAN IPv4 address    → 10.0.1.1
Enter subnet bit count        → 24
Enter upstream gateway        → (leave blank, press Enter)
Configure IPv6 on LAN?        → n
```

### OPT1 (em2) - Server network

```
Select interface → 3

Enter new OPT1 IPv4 address   → 10.0.2.1
Enter subnet bit count        → 24
Enter upstream gateway        → (leave blank, press Enter)
Configure IPv6 on OPT1?       → n
```

**Note:** Static IPs (AD at `10.0.2.10`, Wazuh at `10.0.2.20`) are set on each machine directly — they sit outside the DHCP range so there's no conflict.

### WAN (em0)

WAN was left on DHCP - OPNsense gets an IP automatically from the host network (VirtualBox NAT or bridged adapter).


## Step 3 - Web UI Access

From any machine on the LAN (10.0.1.x), open a browser and go to:

```
https://10.0.1.1
```

Default credentials:
- Username: `root`
- Password: `opnsense`
