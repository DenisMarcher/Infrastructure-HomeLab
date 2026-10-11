# Infrastructure Homelab

This repository contains my infrastructure homelab built on a dedicated VMware ESXi host.

I am adding each part of the lab as I build and test it.

## Current Environment

- VMware ESXi 8.0 Update 3
- Intel Core i3-10100
- 16 GB RAM
- Dedicated physical ESXi host
- ESXi management network on the home LAN
- Internal lab networks isolated on a separate ESXi virtual switch
- Sophos Firewall Home as the permanent routing and security layer
- Separate `SERVERS`, `CLIENTS`, and `OPS` networks routed through Sophos
- Windows Server, Windows 11, and Debian systems used to validate the firewall baseline

## Current Status

- ✅ [ESXi installed and accessible through the Host Client over local network](docs/1-hardware-and-host.md)
- ✅ [ESXi network foundation](docs/2-esxi-networking.md)
- ✅ [Sophos Firewall Home core](docs/3-sophos-firewall.md)
- ⬜ Active Directory and DNS
- ⬜ DHCP and Windows client
- ⬜ Linux and Docker
- ⬜ Monitoring
- ⬜ Backup and restore testing
- ⬜ Vendor firewall lab
- ⬜ Hybrid Microsoft 365 stage
