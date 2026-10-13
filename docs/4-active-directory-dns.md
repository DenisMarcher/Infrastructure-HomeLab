# Active Directory and DNS

I set up `DC01` as the first domain controller for `denyslab.com`. It runs Active Directory Domain Services (AD DS) and DNS on Windows Server 2022. I kept this stage limited to the domain and its core name-resolution services; DHCP, client domain join, and application-specific firewall rules belong to the next stage.

## Requirement

I needed one domain controller with a static address, a new AD forest, and DNS that could resolve both internal domain records and internet names. The server had to remain in the isolated `SERVERS` network behind Sophos Firewall.

## Design

| Setting | Value |
| --- | --- |
| Server | `DC01` — Windows Server 2022 Standard Evaluation |
| AD domain / forest | `denyslab.com` |
| NetBIOS domain | `DENYSLAB` |
| ESXi port group | `SERVERS` (VLAN 20) |
| IPv4 address | `10.77.20.10/24` (static) |
| Default gateway | `10.77.20.1` (`SF01` / `PortA`) |
| Preferred DNS | `10.77.20.10` (the server itself) |
| DNS forwarders | `8.8.8.8`, `1.1.1.1` |
| Forest / domain functional level | Windows Server 2016 |
| Domain controller roles | DNS Server, Global Catalog |

Currently, DC01 is the only domain controller. Active Directory DNS is hosted locally on DC01. Public resolvers are configured as DNS forwarders rather than the server's preferred DNS. The Windows time zone is set to Jerusalem. The NTP server will be assigned during the monitoring configuration phase.

## Implementation

I changed the Windows default hostname to `DC01` before installing AD DS. The server already had a static address on `SERVERS` Zone.

I installed the AD DS role with the management tools and confirmed that Windows reported the role as installed.

![AD DS role installed on DC01](../screenshots/4-ad-ds-role-installed.png)

*AD DS role installation and hostname verification on `DC01`.*

I Deployed a new forest named denyslab.com using the Server Manager promotion wizard. Set both forest and domain functional levels to Windows Server 2016. Enabled DNS Server and Global Catalog roles, and skipped DNS delegation. The prerequisite check passed successfully; completed the promotion and rebooted the server.

![New AD forest settings](../screenshots/4-ad-forest-review.png)

*New forest settings reviewed before promoting `DC01`.*

After promotion, I configured the network adapter to use `10.77.20.10` as its preferred DNS server. I left alternate DNS empty and configured `8.8.8.8` and `1.1.1.1` as forwarders in DNS Manager.

![DC01 IPv4 and DNS settings](../screenshots/4-dc01-ipv4-dns.png)

*Static IPv4 configuration and local DNS on `DC01`.*

## Validation

In Active Directory Users and Computers, `DC01` appears in the `Domain Controllers` container of `denyslab.com` and is shown as a Global Catalog.

![DC01 in Active Directory Users and Computers](../screenshots/4-ad-domain-controller.png)

*`DC01` registered in the domain's `Domain Controllers` container.*

From `DC01`, I checked internal and external name resolution:

- `nslookup dc01.denyslab.com` returned `10.77.20.10`.
- `nslookup google.com` returned public addresses.

![Internal and external DNS lookups](../screenshots/4-dns-lookup.png)

*Internal AD hostname and external DNS lookups from `DC01`.*

I also ran `dcdiag /test:dns /v`. The DNS summary reported passing checks for `DC01`, including the forest-root LDAP SRV record and the `_msdcs.denyslab.com` delegation. The local DNS server and both forwarders passed the reported DNS tests.

![AD DNS test summary](../screenshots/4-dns-dcdiag.png)

*DNS diagnostic summary for `denyslab.com`.*

A separate `net share` check showed both `SYSVOL` and `NETLOGON` shared on `DC01`.

## Troubleshooting and remaining checks

The promotion wizard warned that a DNS delegation could not be created in a parent zone. This was expected for a new forest in a lab environment; otherwise, the prerequisite check passed successfully.
An nslookup check displayed Server: UnKnown for the local IPv6 loopback address (::1), though all queries returned the expected results. This missing display name for the IPv6 loopback resolver was not treated as a DNS lookup failure.

A subsequent dcdiag /q run was not clean, reporting DFSREvent and SystemLog errors. These stemmed from logged events, including a prior unexpected shutdown, Windows Update issues, and DCOM communication errors related to the public DNS addresses. However, both SYSVOL and NETLOGON shares were confirmed present. These event logs will be reviewed during the next health check
## Next stage

The next step involves DHCP deployment and Windows client connectivity. The planned architecture includes running Windows DHCP on DC01, configuring a Sophos DHCP relay for the CLIENTS subnet, and assigning the domain DNS (10.77.20.10) to clients. DHCP scopes, relay policies, inter-zone AD firewall access, and client domain joins have not yet been configured.
