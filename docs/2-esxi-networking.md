# ESXi Networking

## Requirement

ESXi management needs to stay reachable even if the firewall VM is down. The lab also needs three isolated networks for servers, clients, and operations. Traffic between them should cross the firewall.

## Design

Host management stays on `vSwitch0`, outside the virtual firewall. The internal lab networks share a separate vSwitch with no physical uplink.

The [final topology](../diagrams/lab-topology.svg) shows the vSwitches, deployed VMs, and services together.

`vSwitch0` is connected to the home LAN through `vmnic0`. ESXi management remains on this switch through `vmk0` at `10.100.102.69`.

The default `VM Network` port group is now used by `SF01` for its WAN interface. This keeps the firewall connected to the upstream home network without placing ESXi management behind the firewall.

`vSwitch-LAB` has no physical uplink. It carries the three internal lab port groups:

| Port group | VLAN | Subnet | Gateway | Intended use |
| --- | ---: | --- | --- | --- |
| `SERVERS` | 20 | `10.77.20.0/24` | `10.77.20.1` | Infrastructure servers |
| `CLIENTS` | 30 | `10.77.30.0/24` | `10.77.30.1` | Client systems |
| `OPS` | 40 | `10.77.40.0/24` | `10.77.40.1` | Linux, Docker, monitoring, and administration |

The gateway addresses are configured on Sophos Firewall interfaces. ESXi does not have a VMkernel interface on `vSwitch-LAB`.

VLAN tagging is configured on the ESXi port groups. Each internal `SF01` vNIC is attached to one VLAN-backed port group, so ESXi handles the VLAN assignment before traffic reaches the guest. `SF01` does not receive an 802.1Q trunk and does not use VLAN subinterfaces for these networks.

Port groups with the same VLAN ID on a vSwitch would share one broadcast domain. Separate VLAN IDs keep `SERVERS`, `CLIENTS`, and `OPS` isolated at Layer 2 even though the vSwitch has no physical uplink. Routing and access between those networks are handled by Sophos.

- `PortA` → `SERVERS`
- `PortC` → `CLIENTS`
- `PortD` → `OPS`

`PortB` is attached to `VM Network` for WAN access.

## Validation

<p align="center">
  <img src="../screenshots/2-esxi-vswitch-lab-proof.png" alt="vSwitch-LAB configuration in ESXi Host Client" width="100%">
</p>

*ESXi Host Client showing the three lab port groups on `vSwitch-LAB` and no physical adapter attached.*

The firewall deployment and policy validation are documented in [3-sophos-firewall.md](3-sophos-firewall.md).
