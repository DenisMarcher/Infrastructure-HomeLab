# Active Directory and DNS

I installed Active Directory Domain Services and DNS on `DC01` and created the `denyslab.com` forest. This is the first domain controller in the lab.

## Requirement

I needed a domain controller on the `SERVERS` network, with a static IP and DNS for both Active Directory records and internet lookups.

## Design

| Setting | Value |
| --- | --- |
| Server | `DC01`, Windows Server 2022 Standard Evaluation |
| Domain | `denyslab.com` (`DENYSLAB`) |
| Network | `SERVERS`, VLAN 20 |
| Address | `10.77.20.10/24` |
| Gateway | `10.77.20.1` (Sophos `PortA`) |
| Preferred DNS | `10.77.20.10` |
| DNS forwarders | `8.8.8.8`, `1.1.1.1` |
| Domain and forest functional levels | Windows Server 2016 (the highest available on Windows Server 2022) |

`DC01` uses its own DNS service for domain lookups. The public DNS servers are configured as forwarders, not as alternate DNS servers on the network adapter. DNS Server and Global Catalog are enabled on the domain controller.

`denyslab.com` is used as an internal Active Directory DNS zone in this lab. Ownership of the public DNS name has not been confirmed for this documentation update. For a new organizational deployment, I would choose an AD subdomain of a DNS domain the organization owns, such as `ad.<company-domain>`.

## Implementation

I renamed the Windows Server guest to `DC01` (the ESXi VM is named `SRV-DC01`) and installed the AD DS role and management tools.

![AD DS role installed](../screenshots/4-ad-ds-role-installed.png)

*AD DS installed on `DC01`.*

I created a new forest named `denyslab.com` with the Server Manager promotion wizard. I kept the domain and forest functional levels at Windows Server 2016, enabled DNS Server and Global Catalog, and rebooted after promotion.

![New AD forest configuration](../screenshots/4-ad-forest-review.png)

*Forest settings selected during domain controller promotion.*

I set the adapter's preferred DNS server to `10.77.20.10` and configured `8.8.8.8` and `1.1.1.1` as DNS forwarders.

![DC01 network settings](../screenshots/4-dc01-ipv4-dns.png)

*Static IPv4 address and local DNS server on `DC01`.*

## Validation

In Active Directory Users and Computers, `DC01` appears in the `Domain Controllers` container as a Global Catalog.

![Domain controller in ADUC](../screenshots/4-ad-domain-controller.png)

*`DC01` in the `denyslab.com` Domain Controllers container.*

I checked DNS from `DC01`:

- `nslookup dc01.denyslab.com` returned `10.77.20.10`.
- `nslookup google.com` returned public IP addresses.

![Internal and external DNS lookups](../screenshots/4-dns-lookup.png)

*Local domain and external DNS lookups.*

`dcdiag /test:dns /v` passed the reported DNS checks, including the forest-root LDAP SRV record and `_msdcs.denyslab.com` delegation.

![Domain controller DNS test](../screenshots/4-dns-dcdiag.png)

*DNS test results for `DC01`.*

I also checked `net share` and found both `SYSVOL` and `NETLOGON`.

## Troubleshooting

### Warnings after promotion

A separate `dcdiag /q` run reported `DFSREvent` and `SystemLog` entries. I have not confirmed the full details of the related Event Viewer entries for this documentation update. DNS resolution, the later DHCP tests, domain join, and sign-in were validated in subsequent stages. The diagnostic warnings still need a separate review; successful client tests do not prove that all warnings can be ignored.

## Next stage

Windows DHCP, Sophos DHCP relay, and Windows client domain join are covered in [DHCP and Windows Client](5-dhcp-and-windows-client.md).
