# Infrastructure Homelab

I built this lab on one VMware ESXi host to practice network segmentation, Windows domain services, Docker, and monitoring. The host has 16 GB of RAM, and I power on the Windows client for tests.

![Final lab topology](diagrams/lab-topology.svg)

*ESXi management stays on the home network. Sophos routes the three lab networks, and DEB12 runs Docker and Zabbix.*

## Lab setup

- **Host:** ESXi 8.0 Update 3, Intel Core i3-10100, 16 GB RAM
- **Networking:** `vSwitch0` for host management and Sophos WAN; `vSwitch-LAB` for the isolated VLANs

| VM | Network | Address | What runs on it |
| --- | --- | --- | --- |
| `SF01` | WAN, SERVERS, CLIENTS, OPS | WAN `10.100.102.70` (DHCP); internal gateways `10.77.20.1`, `10.77.30.1`, `10.77.40.1` | Sophos Firewall Home |
| `SRV-DC01` (`DC01`) | SERVERS / VLAN 20 | `10.77.20.10` (static) | Windows Server 2022: AD DS, DNS, DHCP, Finance and HR shares |
| `CLIENT-W11-01` | CLIENTS / VLAN 30 | `10.77.30.100` (DHCP at last check) | Windows 11 domain client |
| `DEB12` | OPS / VLAN 40 | `10.77.40.10` (static) | Debian 12, Docker, PostgreSQL, Zabbix Server/Web, Zabbix Agent 2 |

## Build notes

I wrote up each part after setting it up and checking that it worked:

1. [ESXi host](docs/1-hardware-and-host.md) — hardware, installation, management access
2. [ESXi networking](docs/2-esxi-networking.md) — vSwitches, port groups, VLANs
3. [Sophos firewall](docs/3-sophos-firewall.md) — zones, NAT, firewall rules, connectivity tests
4. [Active Directory and DNS](docs/4-active-directory-dns.md) — domain controller and name resolution
5. [DHCP and Windows client](docs/5-dhcp-and-windows-client.md) — DHCP relay, domain join
6. [OUs, GPOs and file access](docs/6-ou-gpo-and-file-access.md) — groups, mapped drive, share permissions
7. [Linux and Docker](docs/7-linux-and-docker.md) — SSH, container storage, Docker Engine and Compose
8. [Zabbix monitoring](docs/8-zabbix-monitoring.md) — Linux and Windows agents, Sophos ICMP checks

## Checks I ran

- Checked internet access and lab-to-home blocking in the Sophos firewall log.
- Joined the Windows 11 client to `denyslab.com`, checked DHCP/DNS, applied GPOs, and tested Finance and HR file access.
- Ran a Docker test container, then started the Zabbix stack with Compose. A [sanitized Compose example](configs/zabbix/compose.yaml) is included.
- Collected Linux and Windows metrics in Zabbix and checked the Sophos OPS interface by ICMP.

The screenshots and troubleshooting details are in the linked notes.

## Scope and open checks

- One ESXi host, one domain controller, and Windows file shares on `DC01`. There is no redundancy.
- A Sophos configuration export is stored off-host. Full system and data restore tests were not performed.
- The last Debian check showed 91% usage on the 2.2 GB root filesystem.
- The initial Zabbix admin password was unchanged at the last check. Some firewall-rule scopes and Zabbix alert delivery remain unverified.
- Windows Server is an evaluation build. `denyslab.com` is an internal AD DNS zone; ownership of the public domain was not verified.
