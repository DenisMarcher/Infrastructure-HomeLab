# ESXi Networking

This is the ESXi network layout before the first virtual machine is deployed.

![Current ESXi network layout](../diagrams/2-esxi-network-layout.svg)

`vSwitch0` remains connected to the home LAN through `vmnic0`. ESXi management stays on this switch through `vmk0` at `10.100.102.69`. The default `VM Network` port group is still present but is not currently used.

`vSwitch-LAB` has no physical uplink. It contains the three internal port groups for the lab:

| Port group | VLAN | Planned subnet | Intended use |
| --- | ---: | --- | --- |
| `SERVERS` | 20 | `10.77.20.0/24` | Infrastructure servers |
| `CLIENTS` | 30 | `10.77.30.0/24` | Client systems |
| `OPS` | 40 | `10.77.40.0/24` | Monitoring, backup, and administration |

The subnets are part of the addressing plan. No lab gateway interfaces or guest IP addresses exist yet.

No VMkernel interface is attached to `vSwitch-LAB`, so ESXi management remains on `vSwitch0` and does not depend on the future virtual firewall.

## ESXi Host Client

![vSwitch-LAB configuration in ESXi Host Client](../screenshots/2-esxi-vswitch-lab-proof.jpg)

*ESXi Host Client showing the three lab port groups on `vSwitch-LAB` and no physical adapter attached.*
