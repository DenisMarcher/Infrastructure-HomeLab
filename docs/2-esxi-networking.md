# ESXi Networking

Before deploying any virtual machines, I separated ESXi management from the internal lab networks.

![ESXi network layout](../diagrams/2-esxi-network-layout.svg)

`vSwitch0` remains connected to the home network through `vmnic0`. It carries the ESXi management network and the default `VM Network`, which will later be used for the firewall WAN interface.

`vSwitch-LAB` has no physical uplink. It is used only for the internal lab networks that will sit behind the virtual firewall.

## Internal networks

| Network | VLAN | Planned subnet | Purpose |
| --- | ---: | --- | --- |
| `SERVERS` | 20 | `10.77.20.0/24` | Infrastructure servers |
| `CLIENTS` | 30 | `10.77.30.0/24` | Windows client systems |
| `OPS` | 40 | `10.77.40.0/24` | Monitoring, backup, and administrative services |

The subnets are part of the addressing plan for the guest systems and firewall interfaces. No guest IP addresses are configured yet because no virtual machines have been deployed.

ESXi management remains on `vSwitch0`, so the host stays reachable even if the virtual firewall is stopped or misconfigured.

At this stage, the ESXi network foundation is in place. Routing and firewall policy will be added with Sophos Firewall Home in the next stage.
