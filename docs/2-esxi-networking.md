# ESXi Networking

Before deploying any virtual machines, I separated ESXi management from the networks that will later sit behind the virtual firewall.

The host started with the default `vSwitch0`, one physical uplink (`vmnic0`), `Management Network`, and `VM Network`. The physical link is up at 1 Gbps full duplex, and ESXi management remains reachable on `10.100.102.69`.

## Lab switch

I created a second standard switch named `vSwitch-LAB` with no physical uplinks.

That is intentional. The internal lab networks should not have a direct path to the home LAN. Once the firewall is deployed, it will be the routing and security boundary between the lab and the outside network.

The current layout is:

```text
vmnic0
  |
vSwitch0
├── Management Network
│   └── ESXi vmk0 - 10.100.102.69
└── VM Network
    └── future firewall WAN

vSwitch-LAB
├── SERVERS  - VLAN 20
├── CLIENTS  - VLAN 30
└── OPS      - VLAN 40
```

No VMkernel adapters were added to `vSwitch-LAB`. ESXi management stays on `vSwitch0`, so the host remains reachable even if the virtual firewall is stopped or misconfigured.

## Internal networks

| Port group | VLAN | Planned subnet | Purpose |
| --- | ---: | --- | --- |
| `SERVERS` | 20 | `10.77.20.0/24` | Domain controller, file services, and other server workloads |
| `CLIENTS` | 30 | `10.77.30.0/24` | Windows client systems |
| `OPS` | 40 | `10.77.40.0/24` | Monitoring, backup, and administrative services |

The subnets are reserved for the next stages. No guest IP addresses have been configured yet because no VMs have been deployed.

The three port groups share the same internal vSwitch but use separate VLAN IDs. The first firewall build will use a separate virtual NIC for each internal network, so there is no need to start with a VLAN 4095 trunk.

## Current state

After creating the lab switch and port groups, the ESXi Host Client remained reachable over the existing management network. `vSwitch0` still has its physical uplink, while `vSwitch-LAB` has none.

Traffic between the lab networks has not been tested yet because there are no guest VMs. That validation will happen as the firewall and first server/client systems are deployed.

The next step is Sophos Firewall Home. Its WAN interface will connect to `VM Network` on `vSwitch0`, while the internal interfaces will connect to `SERVERS`, `CLIENTS`, and `OPS`.
