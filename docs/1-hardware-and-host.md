# ESXi Host

I installed VMware ESXi 8.0 Update 3 on a dedicated PC that runs the infrastructure VMs for this lab.

## Host

- ASUS H410M-A
- Intel Core i3-10100 @ 3.60 GHz
- 16 GB RAM
- 512 GB SSD (the ESXi Host Client showed approximately 334 GB as the VMFS datastore)

## ESXi

Installed version:

- VMware ESXi 8.0 Update 3
- Image profile: `ESXi-8.0U3e-24677879-standard`
- Hostname: `ESXI01.denyslab.com`
- Management IP: `10.100.102.69`
- Gateway: `10.100.102.1`

I manage the host from my main PC through the ESXi Host Client.

After installation, the host clock was incorrect. I corrected the host time using `il.pool.ntp.org` as the NTP server before moving on. Services such as Active Directory and logging depend on accurate time.

![ESXi host summary](../screenshots/1-esxi-host-summary.png)

## VM footprint

With no VMs running, ESXi showed about 14 GB of free memory. I left headroom for the hypervisor and start the Windows client only for tests.

## Next stage

[ESXi Networking](2-esxi-networking.md) covers the vSwitches, port groups, and VLANs.
