# Sophos Firewall Core

Sophos Firewall Home is the permanent routing and security layer for the lab. It runs as `SF01` on ESXi and connects the upstream home network to the three isolated lab networks created in the previous stage.

## Appliance

- Sophos Firewall OS 22.0.2 MR-2 Build 546
- VM name: `SF01`
- 1 vCPU
- 4 GB RAM
- 32 GB system disk
- 80 GB auxiliary disk
- Sophos Firewall Home license

The VM is currently running with one vCPU because the available ESXi licensing/configuration would not allow the planned two-vCPU setting. The firewall workload is intentionally small so the rest of the lab still has enough memory and CPU headroom.

## Interfaces

The firewall uses one virtual NIC per network. No VLAN trunk is configured inside the guest.

| Sophos port | Zone | ESXi port group | Address | Role |
| --- | --- | --- | --- | --- |
| `PortA` | `SERVERS` | `SERVERS` | `10.77.20.1/24` | Server network gateway |
| `PortB` | `WAN` | `VM Network` | `10.100.102.70/24` via DHCP | Upstream home network |
| `PortC` | `CLIENTS` | `CLIENTS` | `10.77.30.1/24` | Client network gateway |
| `PortD` | `OPS` | `OPS` | `10.77.40.1/24` | Operations network gateway |

The interface mapping was verified by matching the virtual NIC MAC addresses in ESXi with the Sophos interfaces.

ESXi management remains on `vSwitch0` at `10.100.102.69` and does not depend on `SF01`.

## Management access

WebAdmin is reachable from the administration PC through `PortB` at `https://10.100.102.70:4444`.

Because `PortB` belongs to the `WAN` zone, HTTPS administration was not enabled broadly for every host on that zone. A local service ACL exception named `FW-ACCESS` permits the required management access from the `ManagementPC` host object.

## Firewall policy

The baseline policy allows the three lab zones to reach the internet while blocking access from the lab to the home LAN by default.

| Rule | Source | Destination | Action |
| --- | --- | --- | --- |
| `LAB_TO_HOME_DENY` | `SERVERS`, `CLIENTS`, `OPS` | WAN zone, `HOME_LAN` network object (`10.100.102.0/24`) | Drop |
| `SERVERS_TO_INTERNET` | `SERVERS` | `WAN` | Accept |
| `CLIENTS_TO_INTERNET` | `CLIENTS` | `WAN` | Accept |
| `OPS_TO_INTERNET` | `OPS` | `WAN` | Accept |

The automatically added MTA rule and the initial `#Default_Network_Policy` were disabled. The built-in final drop rule remains in place.

No allow rules have been added between `SERVERS`, `CLIENTS`, and `OPS` yet.

## NAT

Outbound lab traffic uses the built-in `Default SNAT IPv4` rule. Source addresses are masqueraded through `PortB` before traffic leaves toward the home router.

## Validation

The firewall baseline was tested with one system in each internal network.

| Test system | Network | Internet by IP | DNS/name resolution | Home LAN |
| --- | --- | --- | --- | --- |
| Windows Server | `SERVERS` — `10.77.20.10/24` | Pass | Pass | Blocked |
| Windows 11 client | `CLIENTS` — `10.77.30.10/24` | Pass | Pass | Blocked |
| Debian 12 | `OPS` — `10.77.40.10/24` | Pass | Pass | Blocked |

The Sophos firewall log showed allowed traffic from the internal zones using the corresponding internet rules and `Default SNAT IPv4`. It also showed ICMP from the lab to `10.100.102.1` being denied by `LAB_TO_HOME_DENY`.

Cross-zone ping attempts between the internal test systems were also unsuccessful. Those attempts did not produce a useful firewall record in the captured log, so they are recorded as reachability checks rather than proof of the exact component that discarded ICMP. Inter-zone allow rules are intentionally still absent.

## Troubleshooting

### Traffic to the firewall itself

A system in `SERVERS` initially could not ping its gateway at `10.77.20.1`, even though the interface addressing was correct.

The missing control was Sophos Device Access, which is separate from transit firewall rules. `Ping` and the required DNS service access were enabled for the `SERVERS` zone. The gateway then responded and internet testing could continue.

This distinction matters in Sophos: Device Access controls services addressed to the firewall itself, while firewall rules control traffic passing through it.

### Windows client had IP connectivity but no DNS

The Windows 11 client could ping `8.8.8.8`, which proved that the client gateway, Sophos rule, SNAT, and WAN path were working. Hostname lookups and browser access still failed because the client did not have a usable DNS server configured.

Temporary public DNS was configured for this validation stage. This is not the final domain-client design. After the domain controller is deployed, domain clients will use the domain DNS service.

### Interface discovery during initial setup

The first console view exposed fewer interfaces than expected, while the completed WebAdmin interface list showed `PortA` through `PortD`. Before assigning the final networks, the Sophos ports were matched to the ESXi virtual NICs by MAC address.

## Configuration export

A full Sophos Firewall configuration export was saved on the main administration PC after the baseline validation.

The export is kept off-repository because firewall backups may contain sensitive configuration data. A restore test has not been performed yet, so this export is documented as a saved configuration copy rather than a verified recovery procedure.
