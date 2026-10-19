# DHCP and Windows Client

I configured Windows DHCP on `DC01` for the `CLIENTS` and `OPS` networks, with Sophos Firewall acting as the DHCP relay. I then joined a Windows 11 VM to `denyslab.com` and tested a domain account sign-in.

## Requirement

The two networks needed DHCP from the Windows Server in `SERVERS`. Windows clients also needed the domain DNS server and access to `DC01` for domain join, without opening unrestricted traffic between the lab networks.

## Design

`DC01` keeps the static address `10.77.20.10`. I used Windows DHCP for the scopes and DNS options, and Sophos to relay DHCP requests across the network boundaries.

| Network | DHCP pool | Gateway | DNS |
| --- | --- | --- | --- |
| `CLIENTS` (`10.77.30.0/24`) | `10.77.30.100–199` | `10.77.30.1` | `10.77.20.10` |
| `OPS` (`10.77.40.0/24`) | `10.77.40.100–199` | `10.77.40.1` | `10.77.20.10` |

Both scopes use `denyslab.com` as the DNS domain (DHCP option 015). I reserved the lower part of each subnet for static addressing. `SERVERS` does not use DHCP.

## Implementation

### Windows DHCP

I installed the DHCP Server role on `DC01` and created the `CLIENTS` and `OPS` scopes. Both scopes are active, with eight-day leases.

![Windows DHCP scopes](../screenshots/5-windows-dhcp-scopes.png)

*`CLIENTS` address pool and scope options (003 router, 006 DNS server, and 015 DNS domain). The `OPS` scope is also configured.*

### DHCP relay

I configured Sophos to relay DHCP requests from `PortC` (`CLIENTS`) and `PortD` (`OPS`) to `10.77.20.10`. Sophos itself is not the DHCP server for these networks.

![Sophos DHCP relay](../screenshots/5-sophos-dhcp-relay.png)

*Both relay interfaces point to `DC01`.*

### Access to DC01

I added two firewall rules above the final drop rule:

| Rule | Source zones | Destination | Service |
| --- | --- | --- | --- |
| `CLIENTS_TO_DC01-DNS` | `CLIENTS`, `OPS` | `DC01` (`SERVERS`) | DNS |
| `CLIENTS_TO_DC01-AD` | `CLIENTS` | `DC01` (`SERVERS`) | `AD Services` |

Both rules target `DC01` rather than the full `SERVERS` subnet. The DNS rule also serves OPS; AD service access is limited to CLIENTS.

![Sophos DNS and AD rules](../screenshots/5-sophos-firewall-ad-rules.png)

*Separate DNS and AD service rules to `DC01`.*

I created the `AD Services` service object with TCP 88, 389, 445, 135, 464, and 49152–65535, plus UDP 88, 389, 464, and 123. DNS uses the separate rule on TCP/UDP 53. The AD rule does not require a known Sophos user because the workstation needs to reach the domain controller before sign-in.

The `AD Services` object does not include TCP 3268 (Global Catalog) or TCP 636 (LDAPS). Domain join and sign-in worked without those ports in this test. I will review access rules if a later service requires them.

![AD Services port settings](../screenshots/5-ad-services-ports.png)

*Ports configured in the Sophos `AD Services` object.*

## Validation

### DHCP and DNS

I changed `CLIENT-W11-01` from a static address to DHCP. Its network settings showed `10.77.30.100/24`, gateway `10.77.30.1`, and `10.77.20.10` as both DHCP and DNS server. The DNS suffix was `denyslab.com`.

The client resolved `dc01.denyslab.com` to `10.77.20.10` and resolved `google.com` using `DC01`. The refreshed screenshot also shows successful ICMP replies from `google.com`.

![Windows client DNS validation](../screenshots/5-client-dns-validation.png)

*The Windows client uses `DC01` for internal and external DNS lookups; external connectivity also passed.*

### Domain join

I joined `CLIENT-W11-01` to `denyslab.com` and signed in as `DENYSLAB\jackhanma`. Active Directory Users and Computers on `DC01` showed the client computer account in the `Computers` container at that time.

![Windows domain client and computer account](../screenshots/5-client-domain-join.png)

*Domain account sign-in and the client computer object in AD.*

The client test confirms the `CLIENTS` DHCP path. The `OPS` scope and relay are configured; an OPS DHCP lease test is not part of this stage's evidence.

## Troubleshooting

After switching the client to DHCP, it received the expected IP settings but could not resolve `google.com`. DNS from `CLIENTS` and `OPS` to `DC01` was not allowed yet. I added the DNS firewall rule and repeated the lookups. The updated client evidence shows that both the domain controller name and the external name resolved, and `ping google.com` succeeded.

## Next stage

OU structure, security groups, GPOs, and shared folder permissions are covered in [OUs, Group Policy, and File Access](6-ou-gpo-and-file-access.md).
