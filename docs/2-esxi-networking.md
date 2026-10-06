# ESXi Networking

The ESXi network foundation is in place before any virtual machines are deployed.

![Current ESXi network layout](../diagrams/2-esxi-network-layout.svg)

`vSwitch0` is connected to the home network through `vmnic0`. ESXi management stays on this switch through `vmk0`, using `10.100.102.69`. The default `VM Network` port group is still present but is not in use.

`vSwitch-LAB` is isolated from the physical network. It has no uplink and currently contains three port groups:

| Port group | VLAN | Planned subnet | Intended use |
| --- | ---: | --- | --- |
| `SERVERS` | 20 | `10.77.20.0/24` | Infrastructure servers |
| `CLIENTS` | 30 | `10.77.30.0/24` | Client systems |
| `OPS` | 40 | `10.77.40.0/24` | Monitoring, backup, and administration |

The subnets above are the addressing plan for the next stage. No lab gateway interfaces or guest IP addresses exist yet.

No VMkernel interface is attached to `vSwitch-LAB`. ESXi management therefore remains on `vSwitch0` and does not depend on the future virtual firewall.

## ESXi Host Client

The screenshot below is the configuration evidence from the ESXi Host Client. It shows all three lab port groups on `vSwitch-LAB` with VLAN IDs 20, 30, and 40, and confirms that the switch has no physical adapter attached.

![vSwitch-LAB configuration in ESXi Host Client](../screenshots/2-esxi-vswitch-lab-proof.jpg)
