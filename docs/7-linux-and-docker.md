# Linux and Docker

I reused the existing `DEB12` VM for Docker rather than creating a separate Linux server. The ESXi host has 16 GB of RAM, and the firewall and domain controller already use part of that capacity. This stage gave me a Linux host that I can manage remotely and use for containerized lab services.

## Requirement

I needed a Linux host on the isolated lab network, remote administration from my main PC, enough disk space for container data, and a working Docker Engine with Compose. The existing `DEB12` installation had been used for earlier network tests, so I kept it instead of reinstalling Debian.

## Design

| Setting | Value |
| --- | --- |
| ESXi VM | `DEB12` |
| OS | Debian 12 (Bookworm) |
| Network | `OPS`, VLAN 40 |
| IPv4 / gateway | `10.77.40.10/24` / `10.77.40.1` |
| Allocated memory | 2 GB |
| Virtual disk | 22 GB after expansion |
| Docker data mount | `/dev/sda3` (ext4, about 12 GB) at `/var/lib/docker` |

The disk layout was not a single large root partition. Debian had separate filesystems for `/`, `/var`, `/tmp`, and `/home`. After increasing the virtual disk size in ESXi, the root filesystem was still about 2.2 GB because the additional space had not been partitioned.

I kept the existing partitions and created `/dev/sda3` in the unallocated space. Docker data is stored on that filesystem, not on the small `/var` partition. I also moved containerd's data root to `/var/lib/docker/containerd-store`, because this Docker version uses the containerd image store.

## Implementation

### Remote access

I installed and started OpenSSH on `DEB12`. My main Windows PC is on the home LAN, so I added a route to `10.77.40.10` through the Sophos WAN interface (`10.100.102.70`) and created the Sophos rule `MAIN_TO_DEB12_SSH`. This let me use Windows Terminal instead of typing long commands in the ESXi console.

![SSH session from the Windows PC](../screenshots/7-ssh-session.png)

*An SSH session from Windows to `denis@10.77.40.10`, with the Linux network and disk layout visible.*

### Disk layout and container storage

I expanded the virtual disk to 22 GB in ESXi and then created an additional primary partition in the free space. I formatted `/dev/sda3` as ext4 and configured it to mount at `/var/lib/docker` using its UUID in `/etc/fstab`.

Initially, containerd still used its default `/var/lib/containerd` location on the small `/var` filesystem. After checking the effective configuration, I stopped the Docker services, copied the existing containerd data, changed the root path in `/etc/containerd/config.toml`, and started the services again. The effective root then showed `/var/lib/docker/containerd-store`.

I kept the original containerd directory until the new location could be checked. The new Docker filesystem was about 12 GB and mounted separately from `/` and `/var`.

### Docker Engine and Compose

I installed Docker Engine from Docker's Debian repository and installed the Compose plugin. The tested versions were Docker Engine **29.9.1** and Docker Compose **v5.6.0**. The Docker service was active after installation.

The first repository configuration was wrong: I accidentally created `docker.source`, which APT ignored because it was not a supported source-list filename. I replaced it with a valid `docker.list` entry, refreshed the package lists, and confirmed that `docker-ce` had an installation candidate before proceeding.

## Validation

I tested the installation with `docker run --rm hello-world`. Docker downloaded the image and printed `Hello from Docker!`, confirming that the daemon could pull and run a container.

![Docker installation test](../screenshots/7-docker-hello-world.png)

*Docker and Compose versions, active Docker service, the separate data filesystem, and a completed `hello-world` run.*

The Docker host was subsequently used to run the Zabbix stack described in [Zabbix Monitoring](8-zabbix-monitoring.md). That provides a more useful workload test than `hello-world` alone.

## Troubleshooting

### APT could not find Docker packages

`apt` reported that `docker-ce` had no installation candidate. I checked the repository files and found that the source file was named `docker.source`, so APT ignored it. After creating a valid source-list file, package lookup and installation worked.

### The larger virtual disk did not appear under `/`

Increasing the VMDK in ESXi did not expand the Debian partitions. I checked the partition layout before changing it. Rather than moving the extended partition and the existing filesystems, I added `/dev/sda3` in the free space and mounted it separately for Docker data.

### Container images could fill `/var`

`containerd config dump` initially showed `/var/lib/containerd` as the data root. Since `/var` has only about 1.1 GB total, I moved the containerd data root onto the new Docker filesystem and restarted the services. Both Docker and containerd were active afterward.

## Known Limitations

- The root filesystem is still only about 2.2 GB. It was **91% full**, with about **191–210 MB free** during the final checks. Docker's separate filesystem does not fix the small root filesystem; free space must be monitored before further package installations.
- The lab host has 16 GB of RAM. `DEB12` has 2 GB allocated, so I run other VMs only when they are needed for a test.
- `MAIN_TO_DEB12_SSH` worked for remote administration, but its destination was shown as `Any host` in the Sophos rule list. I still need to restrict the rule to `10.77.40.10` and verify its exact service scope.
- No complete backup or restore procedure has been tested. That work is deferred until after the Linux/Docker and monitoring stages.

## Next Stage

I deployed Zabbix Server, PostgreSQL, and the web frontend as Docker containers on this same VM. See [Zabbix Monitoring](8-zabbix-monitoring.md).
