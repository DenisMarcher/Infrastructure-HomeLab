# ESXi Host

I installed VMware ESXi 8.0 Update 3 on a dedicated PC that will run the infrastructure VMs for this lab.

## Host

- ASUS H410M-A
- Intel Core i3-10100 @ 3.60 GHz
- 16 GB RAM
- 512 GB SSD

The machine was originally planned with 32 GB of RAM, but one motherboard RAM slot is not working. I decided to continue with 16 GB and adjust the VM sizes instead of delaying the lab.

I may add another 256 GB SSD later if I need more room for temporary VMs and vendor firewall labs.

## ESXi

Installed version:

- VMware ESXi 8.0 Update 3
- Image profile: `ESXi-8.0U3e-24677879-standard`
- Hostname: `ESXI01.denyslab.com`
- Management IP: `10.100.102.69`
- Gateway: `10.100.102.1`

I manage the host from my main PC through the ESXi Host Client.

After installation, the host clock was incorrect. I corrected the time configuration before moving on by using `time.cloudflare.com services`. Services such as Active Directory and logging depend on accurate time.

The screenshot below shows the host after the initial setup, with the hostname, ESXi version, hardware information, management IP, and corrected host time visible in the Host Client.

![ESXi Host Client host summary](../screenshots/esxi/esxi-host-summary.png)

*ESXi Host Client after the initial host setup.*

## Resource limits

With no VMs running, ESXi shows about 14 GB of memory free. I am not planning to allocate all of it to guests because the hypervisor and VM overhead still need headroom.

The core VMs will stay small. Larger temporary systems, such as a Windows client or a vendor firewall VM, will only run when I need them for a specific test.

The next step is to build the ESXi network layout while keeping the management connection independent from the virtual firewall.
