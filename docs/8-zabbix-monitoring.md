# Zabbix Monitoring

I deployed Zabbix on `DEB12` to collect basic monitoring data from the Linux host, the Windows domain controller, and Sophos Firewall. Running it in Docker avoided the memory cost of another VM on the 16 GB ESXi host.

## Requirement

I wanted one monitoring interface for the lab, with host availability and operating-system metrics where an agent is available. The initial scope was deliberately small: one Linux host, one Windows Server, and an ICMP check for the firewall.

## Design

The stack runs on `DEB12` (`10.77.40.10`, `OPS`, VLAN 40) using Docker Compose. The deployed images shown in the checks were:

| Compose service | Image | Purpose |
| --- | --- | --- |
| `db` | `postgres:16-alpine` | Zabbix database |
| `zabbix-server` | `zabbix/zabbix-server-pgsql:alpine-7.0-latest` | Data collection and triggers |
| `web` | `zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest` | Web interface |

The PostgreSQL data uses a persistent Docker volume. The Compose configuration is kept in `/home/denis/zabbix`; its `.env` file holds the database password and is **not** part of this repository.

I bound the web interface to `127.0.0.1:8080` on Debian and opened it from the main PC through an SSH tunnel. This avoids publishing the Zabbix web port to the home LAN or internet. The Zabbix Server and database communicate within the Docker network.

For data collection, the Zabbix Server container reaches the Linux agent on the Docker host and the Windows agent across Sophos. Sophos is monitored using ICMP instead of a Zabbix agent.

## Implementation

### Zabbix stack

I started PostgreSQL, Zabbix Server 7.0 LTS, and the Nginx-based web frontend with Docker Compose. The first startup exposed a PostgreSQL connection-limit problem, described in Troubleshooting below. After changing `max_connections` from 30 to 100, the web interface loaded and data collection started.

I used the SSH connection from the main PC to forward the local Zabbix web port:

```powershell
ssh -N -L 8080:127.0.0.1:8080 denis@10.77.40.10
```

The local browser then opened `http://localhost:8080`.

### Monitored hosts

| Zabbix host | Target | Collection method | Evidence |
| --- | --- | --- | --- |
| `Debian 12 - Docker Host` | `10.77.40.10` | Zabbix Agent 2, Linux template | Agent available; 156 configured items at the host-list check |
| `Windows Server 2022 - DC01` | `10.77.20.10` | Zabbix Agent, Windows template | Agent available; 113 items at the host-list check |
| `Sophos Firewall` | `10.77.40.1` | ICMP Ping template | Three ICMP items reporting data |

For Debian, I installed `zabbix-agent2` on the host rather than in another container. The first installation came from the Debian repository and gave version 6.0.14. After correcting root's `PATH` and installing the official Zabbix 7.0 repository package, I updated the agent to **7.0.32**. I configured it to allow requests from the Docker network (`172.18.0.0/16` at the time of the check), restarted the service, and confirmed that it was listening on TCP 10050. The Zabbix host changed from unavailable to available.

For DC01, I installed the Windows agent and linked the Windows monitoring template. Sophos has an `OPS` to `SERVERS` rule named `ZABBIX_TO_DC01`, with source `DEB12` and destination `DC01`, for the agent connection. Zabbix showed the DC01 agent as available and began collecting Windows metrics. The exact TCP service definition of that rule still needs to be checked against port 10050.

For Sophos, I used the interface address **`10.77.40.1`**. The firewall host initially had the wrong target address (`10.77.20.10`, which belongs to DC01). I corrected the host interface before taking the final ICMP evidence.

![Sophos monitoring access rule](../screenshots/8-sophos-zabbix-firewall-rule.png)

*The Sophos `ZABBIX_TO_DC01` rule permits selected traffic from DEB12 in OPS to DC01 in SERVERS. The service's exact port selection still needs review.*

![Zabbix Agent 2 on Debian](../screenshots/8-zabbix-agent-deb12.png)

*The Debian agent was updated to version 7.0.32, started, and found listening on port 10050.*

## Validation

The final host list showed the Linux and Windows agents available. The Sophos host uses ICMP, so its grey ZBX badge is not an agent failure.

![Monitored Zabbix hosts](../screenshots/8-zabbix-hosts.png)

*The three configured hosts and their linked templates; Zabbix agents are available on Debian and DC01.*

The combined **Latest data** screen showed **243 items with data** at the time of capture. It included live Windows disk statistics and items from all three hosts. Another 29 items were listed without data at that moment; these have not all been investigated.

![Zabbix latest data](../screenshots/8-zabbix-latest-data.png)

*Collected data across Linux, Windows Server, and Sophos.*

After correcting the Sophos target address, the ICMP template reported **Up (1)**, **0% packet loss**, and **0.19 ms response time** in the captured check.

![Sophos ICMP validation](../screenshots/8-zabbix-sophos-icmp.png)

*Live ICMP data for the Sophos OPS interface at `10.77.40.1`.*

I also checked the containers and resource use after the monitoring setup. PostgreSQL and the web container showed **healthy** status, and Zabbix Server was running. At the time of capture, memory usage was approximately **100 MiB** for PostgreSQL, **91 MiB** for the web container, and **59 MiB** for Zabbix Server. Debian showed about **1.2 GiB available RAM**. The Docker data filesystem used **918 MiB of 12 GB**; the root filesystem remained at **91% usage**.

![Zabbix containers and resource usage](../screenshots/8-zabbix-containers-resources.png)

*Container status, one-time resource measurements, and remaining Debian disk space.*

## Troubleshooting

### PostgreSQL rejected new client connections

The Zabbix web page displayed a database connection error. PostgreSQL logs repeatedly reported `FATAL: sorry, too many clients already`, and the Zabbix Server logs showed failed database connections.

![PostgreSQL connection-limit error](../screenshots/8-zabbix-postgres-error.png)

*PostgreSQL and Zabbix Server logs during the failed initial connection attempts.*

My Compose setup had PostgreSQL `max_connections` set to 30. I raised it to **100** and applied the Compose changes without removing the database volume. After that change, the Zabbix web interface worked and host data appeared.

### Debian agent was unavailable

The built-in Zabbix host originally pointed to `127.0.0.1:10050`, which refers to the Zabbix container itself, not the Debian host. I installed and configured Agent 2 on Debian, allowed the Docker network as a permitted server source, and changed the monitored interface to `10.77.40.10:10050`. The agent availability indicator then turned green.

### Sophos ICMP target was incorrect

The first Sophos host entry used `10.77.20.10`, the address of DC01. I changed it to the Sophos OPS interface (`10.77.40.1`) and captured fresh ICMP results against the correct address.

## Known Limitations

- Only basic ICMP availability is monitored for Sophos. SNMP and detailed firewall performance metrics have not been configured.
- The Zabbix administrator password was not changed from its initial value. I restricted the web UI to the SSH-forwarded loopback listener, but the default password remains a security issue to fix before expanding access.
- The service definition used by `ZABBIX_TO_DC01` still needs verification to make sure the rule permits **only TCP 10050**.
- The monitoring stack shares a 2 GB Debian VM with Docker and runs without a verified backup. The root filesystem is also close to full.
- I have not tested alert delivery, automatic recovery, or a full Zabbix database restore. The 29 items without data in the combined view have not all been reviewed.

## Next Stage

The next planned work is the Microsoft 365 / Entra ID lab. Backup and restore testing, including protection of the monitoring database and Docker volumes, remains a separate pending stage.
