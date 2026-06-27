
## VLAN 1

### IP addressing plan 


|Device|IP|
|---|---|
|OPNsense|10.0.1.1|
|Windows|10.0.1.10|
|Kali|10.0.1.20|
 
- Name: `vlan1`
- Subnet: `10.0.1.0/24`
### Windows 10 VM


#### Adapter:

- Internal Network → `vlan1`
#### config:

Control Panel → Network and Internet →  Network and sharing center →  Right-click Ethernet →  Properties →  IPv4
##### IP configuration:
- IP address: `10.0.1.10`
- Subnet mask: `255.255.255.0`
- Default gateway: `10.0.1.1`
##### DNS configuration: 
- Preferred DNS: `10.0.2.10` (AD server)
- Alternate DNS: `1.1.1.1` (optional fallback)
### Kali Linux VM

#### Adapter:

- Internal Network → `vlan1`
#### IP addr config:
- edit:
```
sudo nano /etc/network/interfaces
```
- add:

```
auto eth0
iface eth0 inet static
   address 10.0.1.20
   netmask 255.255.255.0
   gateway 10.0.1.1
   dns-nameservers 10.0.2.10 1.1.1.1
```
- restart:

```
 sudo systemctl restart networking
```

##  VLAN 2 
### IP addressing plan 


|Device|IP|
|---|---|
|OPNsense GW|10.0.2.1|
|AD|10.0.2.10|
|Wazuh|10.0.2.20|

- Name: `vlan2`
- Subnet: `10.0.2.0/24`
### Active Directory VM
#### Adapter:
- Internal Network → `vlan2`

#### IP addr config:

Control Panel → Network and Internet → Network and sharing center → Right-click Ethernet → Properties → IPv4

- IP address: `10.0.2.10`
- Subnet mask: `255.255.255.0`
- Gateway: `10.0.2.1`
- DNS:
   - Primary: `10.0.2.10` (itself)
<img width="894" height="633" alt="image" src="https://github.com/user-attachments/assets/140cedbc-130c-40f6-9200-c84aa225482c" />

### Wazuh VM
#### Adapter:
- Internal Network → `vlan2`
#### IP addr config:
- edit:
```
sudo nano /etc/netplan/50-cloud-init.yaml
```
- add:
```YAML
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 10.0.2.20/24
      routes:
        - to: default
          via: 10.0.2.1
      nameservers:
        addresses:
          - 10.0.2.10
          - 1.1.1.1

```

```
sudo netplan apply
```



## OPNsense

### Network adapters:

|Adapter|Type|Network|
|---|---|---|
|NIC 1|NAT NETWORK|Internet|
|NIC 2|Internal Network|vlan1|
|NIC 3|Internal Network|vlan2|




