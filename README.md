# Infrastructure Homelab

I built this homelab on a dedicated VMware ESXi host to practice building, testing, and troubleshooting a small infrastructure environment from the hypervisor up. I document each stage after I configure and verify it.

## Current Status

Completed:

- [ESXi host setup](docs/1-hardware-and-host.md)
- [ESXi network foundation](docs/2-esxi-networking.md)
- [Sophos Firewall core](docs/3-sophos-firewall.md)

Next:

- Active Directory Domain Services and DNS on `DC01`

Planned after that:

- DHCP and domain-client testing
- OU, users, groups, GPO, and file access
- Linux and Docker services
- Monitoring
- Backup and restore testing
- A separate vendor firewall lab
- Microsoft 365 / Entra hybrid work

## Current Lab

- VMware ESXi 8.0 Update 3 on a dedicated physical host
- Intel Core i3-10100 and 16 GB RAM
- ESXi management kept on the home LAN and independent from the virtual firewall
- `vSwitch-LAB` with no physical uplink
- `SERVERS` — VLAN 20 — `10.77.20.0/24`
- `CLIENTS` — VLAN 30 — `10.77.30.0/24`
- `OPS` — VLAN 40 — `10.77.40.0/24`
- Sophos Firewall Home (`SF01`) as the permanent router and firewall for the lab
- Windows Server, Windows 11, and Debian 12 systems used to validate the firewall baseline

## What I Tested

- ESXi management stays reachable without depending on `SF01`.
- `SERVERS`, `CLIENTS`, and `OPS` can reach the internet through Sophos.
- Outbound traffic is source-NATed through the Sophos WAN interface.
- Lab traffic to the home LAN is blocked by an explicit firewall rule.
- Firewall logs show both allowed internet traffic and denied lab-to-home traffic.
- A Windows client DNS failure was isolated from the firewall/NAT path and fixed with temporary public DNS for this stage.

The detailed firewall configuration, validation screenshots, and troubleshooting notes are in [docs/3-sophos-firewall.md](docs/3-sophos-firewall.md).
