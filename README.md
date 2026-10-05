# Infrastructure Homelab

This repository documents a staged infrastructure homelab built on a dedicated VMware ESXi host.

The project is being developed incrementally. Completed stages are documented only after they have been configured and validated in the lab.

## Current Environment

- VMware ESXi 8.0 Update 3
- Intel Core i3-10100
- 16 GB RAM
- Dedicated physical ESXi host
- ESXi management network on the home LAN
- Additional Windows, Linux, firewall, monitoring, and backup components will be added in later stages

## Current Status

- ✅ [ESXi installed and accessible through the Host Client over local network](docs/1-hardware-and-host.md)
- ⬜ ESXi networking
- ⬜ Core virtual firewall
- ⬜ Active Directory and DNS
- ⬜ DHCP and Windows client
- ⬜ Linux and Docker
- ⬜ Monitoring
- ⬜ Backup and restore testing
- ⬜ Vendor firewall lab
- ⬜ Hybrid Microsoft 365 stage
