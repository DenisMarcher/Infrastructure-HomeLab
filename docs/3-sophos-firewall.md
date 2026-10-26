# Sophos Firewall Core

I deployed Sophos Firewall Home to route traffic between the three lab networks and control access to the home network and internet.

## Requirement

For this stage I needed the firewall to:

- route the `SERVERS`, `CLIENTS`, and `OPS` networks
- provide controlled internet access through the home connection
- block lab access to the home LAN by default
- keep ESXi management independent from the firewall VM
- allow WebAdmin access from my main PC without exposing it to the whole WAN zone

The initial policy had no general inter-zone allow rules. I added service-specific rules later for [AD and DNS](5-dhcp-and-windows-client.md) and [Zabbix](8-zabbix-monitoring.md).

## Design

ESXi management stays on `vSwitch0` at `10.100.102.69`. The firewall WAN interface also connects to the home network through `vSwitch0`, while the three internal interfaces connect to the isolated `vSwitch-LAB`.

VLAN tagging is handled by ESXi on the port groups (VLAN IDs 20, 30, and 40; see [2-esxi-networking.md](2-esxi-networking.md)). Each `SF01` vNIC is attached to one of them, so Sophos does not use an 802.1Q trunk or VLAN subinterfaces.

### SF01 VM

- Sophos Firewall OS 22.0.2 MR-2 Build 546
- 1 vCPU
- 4 GB RAM
- 32 GB system disk
- 80 GB auxiliary disk
- 4 virtual NICs

![SF01 virtual machine resources and four ESXi network adapters](../screenshots/3-sophos-vm-hardware.png)

*SF01 VM resources and ESXi network assignments.*

### Interface and zone mapping

| Sophos port | Zone | ESXi port group | VLAN | Address | Role |
| --- | --- | --- | ---: | --- | --- |
| `PortA` | `SERVERS` | `SERVERS` | 20 | `10.77.20.1/24` | Server network gateway |
| `PortB` | `WAN` | `VM Network` | — | `10.100.102.70/24` via DHCP | Upstream home network |
| `PortC` | `CLIENTS` | `CLIENTS` | 30 | `10.77.30.1/24` | Client network gateway |
| `PortD` | `OPS` | `OPS` | 40 | `10.77.40.1/24` | Operations network gateway |

![Final Sophos interface assignments and IP addresses for PortA through PortD](../screenshots/3-sophos-interfaces.png)

*Final interface assignments: `PortA/SERVERS`, `PortB/WAN`, `PortC/CLIENTS`, and `PortD/OPS`.*

![Sophos zones showing WAN, SERVERS, CLIENTS, and OPS membership and device access](../screenshots/3-sophos-zones.png)

*Final zone membership and device-access settings for the lab interfaces.*

The three lab networks are directly connected to `SF01`, so they do not need separate static routes. Firewall policy determines whether traffic is allowed between those connected networks.

During the initial bootstrap, `PortA` temporarily used `172.16.16.16/24`. Before assigning the final networks, I matched the Sophos ports to the ESXi vNICs by MAC address. This prevented a port-group mapping mistake when the four interfaces were reassigned.

## Implementation

### Firewall policy

I placed the lab-to-home deny rule above the internet rules so traffic for `10.100.102.0/24` is dropped before a general WAN allow rule can match it.

| Rule | Source | Destination | Service | Action |
| --- | --- | --- | --- | --- |
| `LAB_TO_HOME_DENY` | `SERVERS`, `CLIENTS`, `OPS` | `WAN` / `HOME_LAN` (`10.100.102.0/24`) | Any | Drop |
| `SERVERS_TO_INTERNET` | `SERVERS` | `WAN` | Any | Accept |
| `CLIENTS_TO_INTERNET` | `CLIENTS` | `WAN` | Any | Accept |
| `OPS_TO_INTERNET` | `OPS` | `WAN` | Any | Accept |

Logging is enabled on these rules. I disabled the automatically added MTA rule and the original `#Default_Network_Policy`. Unmatched traffic still falls to the built-in final `Drop all` rule.

![Sophos firewall rule order with lab-to-home deny and separate internet rules](../screenshots/3-sophos-firewall-rules.png)

*Firewall rules before the later service-specific exceptions.*

No rule currently permits general traffic between `SERVERS`, `CLIENTS`, and `OPS`.

### NAT

Outbound lab traffic uses `Default SNAT IPv4`. The source address is masqueraded and traffic leaves through `PortB`.

![Sophos NAT rules showing Default SNAT IPv4 using MASQ through PortB](../screenshots/3-sophos-nat-rules.png)

*`Default SNAT IPv4` provides outbound masquerading through `PortB`.*

The older linked NAT entries remain visible because Sophos created them with the original default policies. Their parent firewall policies are disabled.

### Management access

I administer WebAdmin from my main PC at `10.100.102.70:4444`. Because `PortB` belongs to the `WAN` zone, I did not enable WebAdmin for the whole zone. A local service ACL exception named `FW-ACCESS` limits this management path to the `ManagementPC` host object.

Device access on the internal zones is also kept narrow. The final zone configuration allows `Ping/Ping6` on `SERVERS` and `CLIENTS`; `OPS` allows HTTPS and `Ping/Ping6`. The `FW-ACCESS` exception permits HTTPS and `Ping/Ping6` from `ManagementPC` in the `WAN` zone.

![Sophos Device Access settings and the FW-ACCESS local service ACL exception](../screenshots/3-sophos-device-access-acl.png)

*Local firewall access is limited by zone-level Device Access settings and the `FW-ACCESS` exception for the management PC.*

## Validation

I tested the baseline with one system in each internal network. Public DNS was used temporarily for these checks. Domain clients use the domain DNS service after `DC01` was deployed.

| Test system | Network and address at initial test | Internet by IP | DNS resolution | Home LAN reachability |
| --- | --- | --- | --- | --- |
| Windows Server | `SERVERS` — `10.77.20.10/24` | Pass | Pass | No reply |
| Windows 11 | `CLIENTS` — `10.77.30.10/24` (later DHCP `.100`) | Pass | Pass | No reply |
| Debian 12 | `OPS` — `10.77.40.10/24` | Pass | Pass | No reply |

### Endpoint checks

![Windows Server connectivity checks from the SERVERS network](../screenshots/3-server-validation.png)

*`SERVERS`: internet and DNS checks pass; the home-router ping receives no reply.*

![Windows 11 connectivity checks from the CLIENTS network](../screenshots/3-client-validation.png)

*`CLIENTS`: the client has the expected address and gateway and can reach the internet.*

![Debian 12 connectivity checks from the OPS network](../screenshots/3-ops-validation.png)

*`OPS`: internet and DNS checks pass; home-LAN and cross-zone pings receive no reply.*

### Firewall log

The firewall log provides the policy evidence. It shows internet traffic from the lab allowed by the matching zone rule with `Default SNAT IPv4` applied, and traffic from all three networks to `10.100.102.1` denied by `LAB_TO_HOME_DENY`. I captured the updated log after `CLIENT-W11-01` had moved to DHCP and `SERVERS` was running `DC01`. The source addresses shown are `10.77.20.10`, `10.77.30.100`, and `10.77.40.10`.

![Sophos Log Viewer showing allowed SNAT traffic and denied traffic from all three lab networks to the home gateway](../screenshots/3-sophos-traffic-log.png?v=575c4ee)

*Allowed internet traffic with SNAT and denied lab-to-home traffic in the same log view.*

Cross-zone ICMP checks also timed out. No matching final-drop log entry was captured, so these checks establish reachability, not which rule handled the packets.

### Configuration backup

I exported the Sophos configuration to my main PC, outside the ESXi datastore. The export was not restore-tested.

## Troubleshooting

### Gateway ping failed from SERVERS

The Windows Server test system initially could not ping its gateway at `10.77.20.1`. The interface addressing and transit policy were already correct, so I checked Sophos Device Access.

The cause was that ICMP to the firewall itself was not enabled for the `SERVERS` zone. After I enabled `Ping`, the gateway responded and I continued with the internet and policy tests. This also confirmed the distinction between Device Access, which controls services terminating on the firewall, and firewall rules, which control forwarded traffic.

### Windows client had IP connectivity but no DNS

The Windows 11 client could ping `8.8.8.8`, but hostname lookups and browser access failed with `DNS_PROBE_FINISHED_BAD_CONFIG`. The successful IP test showed that the client gateway, firewall rule, SNAT, and WAN path were working.

I configured `8.8.8.8` and `1.1.1.1` as temporary DNS servers and repeated the test. Name resolution then worked. These public resolvers were only a temporary validation setting; the domain design now uses `DC01` for DNS.
## Next stage

[Active Directory and DNS](4-active-directory-dns.md) covers the next part of the lab.
