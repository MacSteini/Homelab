# Homelab Guide
Ubuntu Server blueprint – all services isolated in LXD containers, no app workloads on bare metal. High-performance design compatible with any Ubuntu Server host.
## Architecture
```text
Bare Metal (Ubuntu Server)
├── LXD
│   ├── Container 1 – Golden Image / Blueprint (cloning base, stop after sanitising)
│   ├── Container 2 – AdGuard Home (LAN DNS)
│   └── Container 3 – Docker apps (isolated app workloads, accessed via IP:PORT)
└── Host services: UFW · LXD · SSH only
```
Network model:

Layer|Interface|Role
-|-|-
Internal bridge|`lxdbr0` on `eth0`|Admin/management – host access, container admin IPs, SSH proxies, LXD Web UI
LAN exposure|`macvlan` on the host LAN interface|container 2 and 3 use `eth1` inside the container for real LAN presence where required
DNS service|container 2 `eth1`|AdGuard Home DNS for router and LAN clients
App access|container 3 `eth1` + port|Docker apps reachable on the LAN and via Tailnet by published `IP:PORT`

## Why this Stack?
- **LXD over Bare Metal (Isolation-First)**: Running Docker directly on your host OS is administrative suicide. Docker pollutes the host’s `iptables` and network interfaces, making it impossible to debug complex VPN or local DNS issues. LXD provides a “Clean Room” for Docker. If a container goes rogue or the Docker engine hangs the kernel network stack, your host SSH and DNS remain operational.
- **macvlan over NAT (Network Transparency)**: Using a standard bridge for DNS is useless. Without `macvlan`, AdGuard Home only sees the host’s IP, not the actual clients. You lose all visibility into which device is making which request. `macvlan` gives your services a real identity on the LAN.
- **AdGuard Home over Pi-hole (Modernity)**: Pi-hole is a legacy standard that requires multiple “sidecar” containers (like `cloudflared`) to handle modern protocols like DNS-over-HTTPS or TLS. AdGuard Home does this natively and offers a superior client-grouping system. A gateway built for 2024 is preferred over legacy standards.
- **Ubuntu over Raspberry Pi OS (Standardisation)**: Ubuntu Server is selected for its modern kernel and industry-standard `netplan` stack. Raspberry Pi OS is excellent for education, but for a professional-grade server, the OS that powers the cloud is required, not one optimized for desktop use cases.
- **Runbooks over Automation (Ownership)**: Automation scripts (Ansible/Terraform) are “Black Boxes”. When they fail due to a kernel update or a locked `dpkg` process, you are lost. By following these runbooks, you type every command. You learn exactly where your config files live and why they are there. If you didn't type it, you don't own it.
## Prerequisites
- **Hardware:** Any x86 or ARM64 machine running Ubuntu Server; a Raspberry Pi 5 with 16GB RAM is an ideal, more-than-sufficient host for this stack.
- **OS:** Ubuntu Server, fresh headless install
- **Network:** Router with DHCP reservation support and configurable DNS server
- **Second client:** Another LAN or Tailnet client with a browser is required for the AdGuard first-run setup; a shell with `curl` and `dig` is useful for verification
- **Account:** Tailscale account if you want to use the optional [Tailscale Runbook](Optional/Tailscale%20Runbook.md "Tailscale Runbook")
## Placeholders
Replace every placeholder with the actual value before running any command in any runbook.
### Core Network Infrastructure

Placeholder|Meaning|Example/Suggested Value|Your Real Data
-|-|-|-
`<HOST_LAN_GW>`|Router gateway IP on the LAN|`192.168.178.1`|
`<HOST_LAN_IP>`|Host LAN IP on the router|`192.168.178.55`|
`<LAN_CIDR>`|Router LAN subnet|`192.168.178.0/24`|
`<LAN_PREFIX_LEN>`|Router LAN prefix length|`24`|
`<FALLBACK_NAMESERVER>`|Fallback nameserver|`1.1.1.1`|
`<LOCAL_DOMAIN>`|Your preferred local homelab domain|`home.arpa`|

### LXD Bridge Infrastructure (lxdbr0)

Placeholder|Meaning|Example/Suggested Value|Your Real Data
-|-|-|-
`<LXDBR0_GW>`|Bridge IP with prefix (host)|`10.10.10.1/24`|
`<LXDBR0_GW_IP>`|Bridge IP without prefix|`10.10.10.1`|
`<LXDBR0_SUBNET>`|Container admin subnet|`10.10.10.0/24`|
`<LXDBR0_PREFIX_LEN>`|Container admin prefix length|`24`|

### Containers Configuration

Placeholder|Meaning|Example/Suggested Value|Your Real Data
-|-|-|-
`<C1_NAME>`|Container 1 name (Blueprint)|`blueprint`|
`<C1_ETH0>`|C1 static IP (on lxdbr0)|`10.10.10.11`|
`<C2_NAME>`|Container 2 name (DNS/VPN)|`gateway`|
`<C2_ETH0>`|C2 static IP (on lxdbr0)|`10.10.10.12`|
`<C2_ETH1>`|C2 LAN IP (from DHCP)|`192.168.178.60`|
`<C3_NAME>`|Container 3 name (Docker)|`docker`|
`<C3_ETH0>`|C3 static IP (on lxdbr0)|`10.10.10.13`|
`<C3_ETH1>`|C3 static LAN IP|`192.168.178.61`|

### SSH & Access

Placeholder|Meaning|Example/Suggested Value|Your Real Data
-|-|-|-
`<SSH_ADMIN_USER>`|Primary SSH login user|`macsteini`|
`<SSH_PUBKEY_PATH>`|Public key path on your PC|`~/.ssh/id_ed25519.pub`|

### Docker Configuration

Placeholder|Meaning|Example/Suggested Value|Your Real Data
-|-|-|-
`<DOCKER_NETWORK>`|Shared Docker network name|`homelab`|
`<DOCKER_ROOT_DIR>`|Bind mounts base path|`/opt/docker`|
`<DOCKER_DATA_ROOT>`|Docker internal data path|`/opt/docker/data`|
`<DOCKER_STACKS_DIR>`|Compose files path|`/opt/docker/stacks`|
`<DOCKER_APP_PORT>`|Example App Port (for testing)|`8080`|

### Application Specific (Late Bind)
These are used within Docker Compose stacks and individual service runbooks:
- `APPNAME`: The name of the application stack.
- `CONTAINER_NAME`: The specific name for a Docker container.
- `VOLUME_NAME`: Named volume for persistent data.
- `REPLACE_WITH_STRONG_DB_PASSWORD`: Placeholder for secure passwords.
- `REPLACE_WITH_ADMIN_USERNAME`: Placeholder for initial admin setups.
- `REPLACE_WITH_STRONG_SECRET_KEY`: Placeholder for JWT or session secrets.
## Build Sequence
### Phase 0 – OS Installation (Optional Initial Step)
> Use this if you are starting with a fresh Raspberry Pi and want to boot Ubuntu Server from NVMe. If you already have a running Ubuntu system, skip to phase 1.

→ [the OS on NVMe Runbook](Optional/OS%20on%20NVMe%20–%20Fresh%20Install%20and%20Headless%20Setup.md "the OS on NVMe Runbook"): Prepare a fresh Ubuntu Server install for headless use, boot the Raspberry Pi from NVMe, create the primary admin user, and finish with a host that is reachable over SSH for the [Foundation Runbook](Essential/Foundation%20Runbook.md "Foundation Runbook").
### Phase 1 – Bare Metal
→ the [Foundation Runbook](Essential/Foundation%20Runbook.md#phase-1---bare-metal "Foundation Runbook"):
1. Configure Netplan (`eth0`, DHCP)
2. Install and initialise LXD (`lxdbr0` bridge, Web UI on `:8443`)
3. Configure fixed router reservation for `<HOST_LAN_IP>`
4. Configure host firewall (UFW) – LAN SSH, LXD Web UI
### Phase 2 – Blueprint (Container 1)
→ the [Foundation Runbook](Essential/Foundation%20Runbook.md#phase-2---blueprint-container-1 "Foundation Runbook"):
1. Launch container
2. Disable cloud-init network management
3. Configure Netplan – static `eth0` only
4. Set hostname and `/etc/hosts`
5. Sanitise: clear machine-id, remove SSH host keys
6. Stop – blueprint is ready to clone
### Phase 3 – DNS (Container 2)
> Before starting this phase, make sure you have another LAN or Tailnet client available for the AdGuard first-run web UI.

→ the [Foundation Runbook](Essential/Foundation%20Runbook.md#phase-3---dns-container-2 "Foundation Runbook"):
1. Clone from blueprint, add `eth1`, start
2. Fix hostname, verify/adapt Netplan (static `eth0`, DHCP `eth1`)
3. Configure DHCP reservation for `eth1` on the router
4. Install AdGuard Home, verify DNS binding on `eth1`
5. Point router DNS to `eth1` LAN IP
### Phase 4 – Docker (Container 3)
→ the [Foundation Runbook](Essential/Foundation%20Runbook.md#phase-4---docker-container-3 "Foundation Runbook"):
1. Clone from blueprint, add `eth1`, start
2. Fix hostname, verify/adapt Netplan (static `eth0`, static `eth1`)
3. Apply LXD nesting prerequisites **on the host** before touching Docker
4. Install Docker, set `data-root`, create `<DOCKER_NETWORK>` network
5. Verify default route via `<HOST_LAN_GW>` on `eth1`

→ the [Docker Runbook](Essential/Docker%20Runbook.md "Docker Runbook"): Learn how to operate and manage your app workloads inside container 3.
### Phase 5 – Validation
→ the [Foundation Runbook](Essential/Foundation%20Runbook.md#phase-5---validation "Foundation Runbook"): Verify bare metal, all containers, routing, DNS, Docker.
### Phase 6 – Remote Access (Optional)
→ the [Tailscale Runbook](Optional/Tailscale%20Runbook.md "Tailscale Runbook"): Establish secure remote access and subnet routing to reach your homelab from anywhere.
### Phase 7 – Local DNS and Reverse Proxy (Optional)
→ the [Caddy Runbook](Optional/Caddy%20Runbook.md "Caddy Runbook"): Configure friendly hostnames and a reverse proxy for your Docker apps.
## Stack at a Glance

Component|Container|Access point
-|-|-
AdGuard Home (DNS)|container 2|`eth1` LAN IP, port 3000 (setup) / 53 (DNS)
Tailscale|container 2|Tailscale IP (if the [Tailscale Runbook](Optional/Tailscale%20Runbook.md "Tailscale Runbook") is used)
Docker apps|container 3|`eth1` LAN IP + published port
LXD Web UI|Bare Metal|`<HOST_LAN_IP>:8443`
SSH|Bare Metal|`<HOST_LAN_IP>:22`

## Runbook Index
### Core build path

Runbook|Scope
-|-
[Foundation Runbook](Essential/Foundation%20Runbook.md "Foundation Runbook")|Bare metal + LXD + UFW + blueprint container + DNS container + Docker container
[Docker Runbook](Essential/Docker%20Runbook.md "Docker Runbook")|Docker operations, troubleshooting, and removal inside container 3

### Optional extensions

Runbook|Scope
-|-
[OS on NVMe – Fresh Install and Headless Setup](Optional/OS%20on%20NVMe%20–%20Fresh%20Install%20and%20Headless%20Setup.md "OS on NVMe – Fresh Install and Headless Setup")|Fresh Ubuntu Server install on NVMe, first boot, and first SSH access
[Tailscale Runbook](Optional/Tailscale%20Runbook.md "Tailscale Runbook")|Secure remote access and subnet routing via container 2
[Caddy Runbook](Optional/Caddy%20Runbook.md "Caddy Runbook")|Reverse proxy and local DNS integration for friendly hostnames