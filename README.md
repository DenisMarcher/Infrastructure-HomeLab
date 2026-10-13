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
- Sophos Firewall Home as the permanent firewall for the lab networks
- Windows Server 2022 running AD DS, DNS, and DHCP on `DC01`
- Additional Windows roles, Linux services, monitoring, and backup components will be added in later stages

## Current Status

- ✅ [ESXi installed and accessible through the Host Client over local network](docs/1-hardware-and-host.md)
- ✅ [ESXi network foundation](docs/2-esxi-networking.md)
- ✅ [Sophos Firewall core](docs/3-sophos-firewall.md)
- ✅ [Active Directory and DNS](docs/4-active-directory-dns.md)
- ✅ [DHCP and Windows client](docs/5-dhcp-and-windows-client.md)
- ✅ [OUs, Group Policy, and file access](docs/6-ou-gpo-and-file-access.md)
- ⬜ Linux and Docker
- ⬜ Monitoring
- ⬜ Backup and restore testing
- ⬜ Vendor firewall lab
- ⬜ Hybrid Microsoft 365 stage
