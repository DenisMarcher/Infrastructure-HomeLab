# ESXi Networking

I built the ESXi network foundation before deploying any virtual machines.

![Current ESXi network layout](../diagrams/2-esxi-network-layout.svg)

`vSwitch0` stays connected to the home LAN through `vmnic0`. ESXi management uses `vmk0` with management IP `10.100.102.69`. The default `VM Network` is currently unused and is reserved for the future firewall WAN side.

`vSwitch-LAB` has no physical uplink. It contains the three internal port groups that will sit behind the virtual firewall.

## Internal networks

| Network | VLAN | Planned subnet | Purpose |
| --- | ---: | --- | --- |
| `SERVERS` | 20 | `10.77.20.0/24` | Infrastructure servers |
| `CLIENTS` | 30 | `10.77.30.0/24` | Windows client systems |
| `OPS` | 40 | `10.77.40.0/24` | Monitoring, backup, and administrative services |

These subnets are part of the addressing plan only. No guest IP addresses or lab gateway interfaces exist yet because the firewall and guest VMs have not been deployed.

No VMkernel interface was added to `vSwitch-LAB`, so ESXi management remains independent from the virtual firewall.

## ESXi configuration

![vSwitch-LAB in ESXi Host Client](../screenshots/2-esxi-vswitch-lab-proof.jpg)

The Host Client view shows `SERVERS` on VLAN 20, `CLIENTS` on VLAN 30, and `OPS` on VLAN 40 attached to `vSwitch-LAB`. It also shows that the switch has no physical adapter attached.
