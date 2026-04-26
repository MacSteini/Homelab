# Foundation Runbook
> Placeholders: [Homelab Guide](../Homelab%20Guide.md#placeholders "Homelab Guide").
## Scope
Build the host foundation plus the three core LXD containers so the homelab is operational: blueprint ready, AdGuard Home working in container 2, and Docker ready in container 3.
## Architecture
- Bare Metal:
	- Ubuntu Server plus:
		- LXD + Web UI
		- Netplan
		- SSH
		- UFW
- Container 1 (Golden Image / Blueprint):
	- Clean base image as a template
	- `eth0`: internal LXD bridge IP
	- `eth1`: added only to clones that actually need LAN visibility or a second uplink
	- Ubuntu Server plus:
		- Netplan
		- Cloud-init network config disabled
- Container 2:
	- Accessed via `eth0`
	- Additionally `eth1`: `macvlan` on the actual LAN, parented from the host LAN interface `eth0`
	- Golden Image plus:
		- AdGuard Home: uses the `eth1` LAN IP in container 2
- Container 3:
	- Accessed via `eth0` and `eth1`
	- `eth0`: internal LXD bridge IP
	- `eth1`: static LAN IP via `macvlan`, parented from the host LAN interface `eth0`
	- Golden Image plus Docker: app workloads can later be exposed on `<C3_ETH1>:PORT` from LAN and Tailnet once deployed via the [Docker Runbook](Docker%20Runbook.md "Docker Runbook")
## Build Order
1. Bare Metal: configure Netplan
2. Bare Metal: install and initialise LXD
3. Router: reserve `<HOST_LAN_IP>` for the host, reserve `<C2_ETH1>` for container 2, and ensure `<C3_ETH1>` is reserved or excluded from the DHCP pool before assigning it statically
4. Router: set `<C2_ETH1>` as the router DNS server for LAN clients
## Phase 1 – Bare Metal
### Create/modify Netplan
Run:
```sh
sudo tee /etc/netplan/01-eth0.yaml >/dev/null <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false
      accept-ra: true
EOF

sudo chmod 600 /etc/netplan/01-eth0.yaml && sudo netplan try && sudo netplan apply && ip a && ip r
```
> If a `*.yaml` file already exists in `/etc/netplan/`, rename it first, e. g., `*.yaml` → `*.yaml.bak`.
### LXD installation and initialisation
Run:
```sh
sudo snap install lxd
sudo lxd init
```
### `lxd init` target state
Use `lxd init` so the resulting state matches this design exactly:
- Bridge: `lxdbr0`
- IPv4 bridge subnet: `<LXDBR0_GW>`
- Container admin subnet on `eth0`: `<LXDBR0_SUBNET>`
- IPv6 as required
- LXD UI/API on `:8443`

Minimum prompt guidance during `sudo lxd init`:
- Create a new local storage pool unless you are deliberately attaching an existing one
- Create the `lxdbr0` bridge during `lxd init`
- Set its IPv4 address/subnet to `<LXDBR0_GW>`
- Choose a bridge subnet that does not overlap with `<LAN_CIDR>`
- Expose the LXD UI/API on `:8443`
### Important consistency rule
- The host bridge `lxdbr0` and the static container `eth0` addresses must use the same IPv4 subnet.
- In this setup:
	- Host bridge `lxdbr0` = `<LXDBR0_GW>`
	- Container admin subnet = `<LXDBR0_SUBNET>`
- Do not change the bridge subnet later without updating the static container Netplan configuration as well.
### Verify
Run:
```sh
sudo lxc info && sudo lxc network list && sudo lxc network show lxdbr0 && ip -4 addr show dev lxdbr0 && sudo lxc storage list && ss -tulpn | grep 8443
```
### Firewall
Only open ports that the host itself requires by default.

Run:
```sh
sudo lxc network set lxdbr0 ipv4.firewall false
sudo lxc network set lxdbr0 ipv6.firewall false

sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow in on lxdbr0 comment "LXD bridge: input"
sudo ufw route allow in on lxdbr0 comment "LXD bridge: routed input"
sudo ufw route allow out on lxdbr0 comment "LXD bridge: routed output"

sudo ufw allow from <LAN_CIDR> to any port 22 proto tcp comment "Host: SSH"
sudo ufw allow from <LAN_CIDR> to any port 8443 proto tcp comment "LXD: Web UI"

sudo ufw enable
sudo ufw status verbose
```
## Phase 2 – Blueprint (Container 1)
### Create and start
Run:
```sh
sudo lxc launch images:ubuntu/24.04 <C1_NAME>
```
> `lxc launch images:ubuntu/24.04 <C1_NAME>` does not start an ISO installer, but an Ubuntu LXD container. On the host, the matching ARM64 variant is pulled automatically.
### Enter the container
Run on the host:
```sh
sudo lxc exec <C1_NAME> -- bash
```
### Disable cloud-init network config
Run inside the container:
```sh
sudo mkdir -p /etc/cloud/cloud.cfg.d && sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null <<'EOF'
network: {config: disabled}
EOF

sudo rm -f /etc/netplan/50-cloud-init.yaml
```
### Create/modify Netplan
Run inside the container:
```sh
sudo tee /etc/netplan/01-eth0.yaml >/dev/null <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - <C1_ETH0>/<LXDBR0_PREFIX_LEN>
EOF

sudo chmod 600 /etc/netplan/01-eth0.yaml && sudo netplan try && sudo netplan apply && ip a && ip r
```
### Fix hostname resolution
Run inside the container:
```sh
sudo hostnamectl set-hostname <C1_NAME> && sudo tee /etc/hosts >/dev/null <<'EOF'
127.0.0.1 localhost
127.0.1.1 <C1_NAME>

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF
```
### Notes
- `eth0` = static internal admin IP on `lxdbr0`
- `lxdbr0` on the host must be `<LXDBR0_GW>`
- `eth0` addresses of cloned containers must stay within `<LXDBR0_SUBNET>`
- Keep the blueprint single-homed by default
- Add `eth1` only to clones that actually need `macvlan`/LAN access
- No default route is configured explicitly on `eth0`
- `50-cloud-init.yaml` must not reappear
### Sanitise the blueprint before cloning
Run inside the container:
```sh
sudo truncate -s 0 /etc/machine-id && sudo rm -f /var/lib/dbus/machine-id
```
### Stop container
Run on the host:
```sh
sudo lxc stop <C1_NAME>
```
## Phase 3 – DNS (Container 2)
### Create from blueprint
Run on the host:
```sh
sudo lxc copy <C1_NAME> <C2_NAME> && sudo lxc config device add <C2_NAME> eth1 nic nictype=macvlan parent=eth0 && sudo lxc start <C2_NAME>
```
### Enter the container
Run on the host:
```sh
sudo lxc exec <C2_NAME> -- bash
```
### Fix hostname resolution
Run inside the container:
```sh
sudo hostnamectl set-hostname <C2_NAME> && sudo tee /etc/hosts >/dev/null <<'EOF'
127.0.0.1 localhost
127.0.1.1 <C2_NAME>

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF
```
### Check/adapt Netplan if required
Run inside the container:
```sh
sudo rm -f /etc/netplan/*.yaml && sudo tee /etc/netplan/01-netcfg.yaml >/dev/null <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - <C2_ETH0>/<LXDBR0_PREFIX_LEN>
    eth1:
      dhcp4: true
      dhcp6: false
EOF

sudo chmod 600 /etc/netplan/01-netcfg.yaml && sudo netplan generate && sudo netplan apply && ip a && ip r
```
### Notes
- `eth0` = static internal admin IP on `lxdbr0`
- Internal subnet: `<LXDBR0_SUBNET>`
- `eth1` = LAN IP via DHCP after router reservation as `<C2_ETH1>`
- Configure a DHCP reservation for `eth1` on the router
- Do not plan to enable IPv6 DNS in this container initially.
- `eth0` must not receive DHCP
- `eth0` must not become the default uplink
- After AdGuard Home is working, the host must use `<C2_ETH0>` as its DNS server
- `<C2_ETH1>` is for router/LAN clients
- Do not make the host depend on router-distributed DNS in this design
### Network logic
- `eth0`:
	- Host → Container
	- SSH/admin
	- Management
	- Attached to `lxdbr0`
	- Internal subnet: `<LXDBR0_SUBNET>`
- `eth1`:
	- Actual LAN IP via router reservation as `<C2_ETH1>`
	- DNS IP for router and clients
	- AdGuard listens here for LAN clients
### Important note
- The host cannot reach the `macvlan` IP of `eth1` directly
- The host always uses the bridge IP on `eth0`
### AdGuard Home
#### Installation
Run inside the container:
```sh
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh
```
#### Reserve port 53 before the setup wizard
Run inside the container:
```sh
sudo ss -tulpn | grep :53 || true
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo ss -tulpn | grep :53 || true
```
> `systemd-resolved` is the common stub-listener conflict. Port `53` must be free before AdGuard Home takes over DNS.
#### Initial setup and config refinement
- Open the web UI from another LAN client or another Tailnet client and complete the initial setup once.
- Do not use the host for this step. The host cannot reach the `macvlan` LAN IP of container 2 directly.
- Copy the generated AdGuard username and password before continuing.
- Stop AdGuard Home.
- Back up the generated live config inside the container.
- Edit the live AdGuard Home YAML config inside the container directly so the generated `users` values are preserved and any later preferences match your real environment.
- Start AdGuard Home again.

Run inside the container:
```sh
CONFIG_FILE="$(find /opt/AdGuardHome -maxdepth 1 -type f | grep -m1 '/opt/AdGuardHome/.*\.yaml$')"
sudo systemctl stop AdGuardHome
sudo cp "$CONFIG_FILE" "$CONFIG_FILE.bak"
sudoedit "$CONFIG_FILE"
sudo systemctl start AdGuardHome
```
#### Afterwards
- Ensure DNS is reachable via `<C2_ETH1>`.
- Ensure it is reachable via the LAN IP.
- Verify the actual listening and binding state with `ss -tulpn`.
- Ensure DNS is reachable via `<C2_ETH0>` from the host as well.
- Ensure AdGuard Home listens on both `<C2_ETH0>:53` and `<C2_ETH1>:53`.
- Do not restrict AdGuard Home DNS binding to `eth1` only.
#### Test
If `dig` is not installed in the container yet, run `sudo apt update && sudo apt install -y dnsutils` first.

Run inside the container:
```sh
sudo ss -tulpn | grep :53 && dig google.com @<C2_ETH0> && dig google.com @<C2_ETH1> && dig test.<LOCAL_DOMAIN> @<C2_ETH1>
```
### Router / DNS
Run on the router:
- Reserve `<HOST_LAN_IP>` for the host
- Obtain `eth1` via DHCP from the router
- Configure a DHCP reservation for `<C2_ETH1>` on the router
- Before setting router DNS to `<C2_ETH1>`, switch the host DNS to `<C2_ETH0>`
- Set `<C2_ETH1>` as the DNS server on the router
- Use only `<C2_ETH1>`
- The host must not rely on the router-distributed DNS in this design
- Do not use the internal LXD bridge IP of `eth0`
- Keep any additional LAN-facing app or management addresses fixed if they must remain reachable by stable `IP:PORT`
## Bare Metal: switch host DNS to container 2 internal DNS
Run on the host:
```sh
sudo tee /etc/netplan/01-eth0.yaml >/dev/null <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false
      dhcp6: false
      accept-ra: false
      nameservers:
        addresses:
          - <C2_ETH0>
          - <FALLBACK_NAMESERVER>
EOF

sudo chmod 600 /etc/netplan/01-eth0.yaml
sudo netplan generate
sudo netplan apply
sudo systemctl restart systemd-resolved
resolvectl flush-caches
resolvectl query ports.ubuntu.com
apt update
```
Notes:
- Only do this after AdGuard Home is working in container 2.
- The host must use `<C2_ETH0>` for DNS, not `<C2_ETH1>`
- `<FALLBACK_NAMESERVER>` remains only as a fallback resolver
## Phase 4 – Docker (Container 3)
### Create from blueprint
Run on the host:
```sh
sudo lxc copy <C1_NAME> <C3_NAME> && sudo lxc config device add <C3_NAME> eth1 nic nictype=macvlan parent=eth0 && sudo lxc start <C3_NAME>
```
### Enter the container
Run on the host:
```sh
sudo lxc exec <C3_NAME> -- bash
```
### Fix hostname resolution
Run inside the container:
```sh
sudo hostnamectl set-hostname <C3_NAME> && sudo tee /etc/hosts >/dev/null <<'EOF'
127.0.0.1 localhost
127.0.1.1 <C3_NAME>

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF
```
### Netplan
Run inside the container:
```sh
sudo rm -f /etc/netplan/*.yaml && sudo tee /etc/netplan/01-netcfg.yaml >/dev/null <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - <C3_ETH0>/<LXDBR0_PREFIX_LEN>
      routes:
        - to: 100.64.0.0/10
          via: <C2_ETH0>
    eth1:
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - <C3_ETH1>/<LAN_PREFIX_LEN>
      routes:
        - to: default
          via: <HOST_LAN_GW>
      nameservers:
        addresses:
          - <C2_ETH1>
          - <FALLBACK_NAMESERVER>
EOF

sudo chmod 600 /etc/netplan/01-netcfg.yaml && sudo netplan generate && sudo netplan apply && ip a && ip r
```
### Notes
- `eth0` = static internal admin IP on `lxdbr0`
- `eth1` = static LAN IP
- Internal subnet: `<LXDBR0_SUBNET>`
- `eth0` must never use DHCP
- `eth1` must never use DHCP in this foundation design
- Ensure the router does not hand out `<C3_ETH1>` to another device
- Use `<C2_ETH1>` as the DNS server in container 3 because AdGuard serves DNS on its LAN-facing interface.
- `/etc/netplan/` must contain exactly one active YAML file for this container
- Default route = `<HOST_LAN_GW>` via `eth1`
- Tailnet return route `100.64.0.0/10` must go via `<C2_ETH0>` on `eth0`
- Verify explicitly that the effective default route matches the intended design
### LXD nesting prerequisites for Docker
Run on the host before installing Docker inside `<C3_NAME>`:
```sh
sudo lxc config set <C3_NAME> security.nesting true && sudo lxc config set <C3_NAME> security.syscalls.intercept.mknod true && sudo lxc config set <C3_NAME> security.syscalls.intercept.setxattr true && sudo lxc restart <C3_NAME>
```
#### Why this is required
- Docker in an LXD system container needs these LXD settings.
- Without them, Docker containers may fail to start with OCI and runtime errors such as access problems around `/proc/sys` or sysctl handling.
- Therefore, set these options immediately after creating `<C3_NAME>` and before the Docker installation.
### Docker
#### Installation
Run inside the container:
```sh
curl -fsSL https://get.docker.com -o install-docker.sh && sudo sh install-docker.sh
```
#### Docker `data-root`
Use `<DOCKER_DATA_ROOT>` as the Docker data root so Docker-managed data, including exception-case named volumes, also lives under `<DOCKER_ROOT_DIR>`.

Run inside the container:
```sh
sudo mkdir -p /etc/docker <DOCKER_DATA_ROOT> && sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "data-root": "<DOCKER_DATA_ROOT>"
}
EOF

sudo systemctl restart docker && sudo docker info | grep "Docker Root Dir"
```
Expected:
```text
Docker Root Dir: <DOCKER_DATA_ROOT>
```
#### Shared Docker network
Create the shared external network only after the new Docker `data-root` is active.

Run inside the container:
```sh
sudo docker network create <DOCKER_NETWORK>
sudo docker network ls | grep <DOCKER_NETWORK>
```
#### Why this matters
- `<DOCKER_DATA_ROOT>` keeps Docker-managed data under `<DOCKER_ROOT_DIR>`.
- Bind mounts remain the default choice where they work cleanly
- Some containers may still fail with bind mounts in Docker-in-LXD due to UID/GID and permission mapping issues
- Use named volumes only for exceptions where bind mounts are not workable or not worth the friction

Container 3 is valid with direct `IP:PORT` access only.
## Phase 5 – Validation
### Bare Metal
Run on the host:
```sh
ip a && ip r && networkctl && resolvectl status && sudo lxc list && sudo lxc network list && sudo lxc network show lxdbr0 && ip -4 addr show dev lxdbr0 && ss -tulpn && ping -c 2 <C1_ETH0> && ping -c 2 <C2_ETH0> && ping -c 2 <C3_ETH0> && getent hosts ports.ubuntu.com
```
### Container 1
Run inside the container before stopping the blueprint:
```sh
ip a && ip r && ss -tulpn && ls -l /etc/netplan && test ! -s /etc/machine-id
```
Expected:
- `/etc/machine-id` is empty
### Container 2
Run inside the container:
```sh
ip a && ip r && ping -c 3 <FALLBACK_NAMESERVER> && ping -c 3 <C3_ETH0> && ss -tulpn && dig google.com @<C2_ETH0> && dig google.com @<C2_ETH1> && dig test.<LOCAL_DOMAIN> @<C2_ETH1>
```
### Container 3
Run inside the container:
```sh
ip a && ip r && ip route get <FALLBACK_NAMESERVER> && sudo docker ps && sudo docker network inspect <DOCKER_NETWORK> && sudo docker info | grep "Docker Root Dir" && sudo docker system df
```
## Target state
### Bare Metal
- Minimal
- Only LXD + UI + firewall
- `lxdbr0` = `<LXDBR0_GW>`
- No app services
### Container 1
- Clean golden image
- Single-homed on `eth0` by default
- Cloud-init network config disabled
- No residual artefacts
- `eth0` = `<C1_ETH0>` within `<LXDBR0_SUBNET>`
- Cloneable at any time
### Container 2
- AdGuard is the central DNS
- `eth0` = `<C2_ETH0>` within `<LXDBR0_SUBNET>`
- `eth1` = `<C2_ETH1>` as the LAN DNS IP
### Container 3
- `eth0` = `<C3_ETH0>` within `<LXDBR0_SUBNET>`
- `eth1` = `<C3_ETH1>` on `<LAN_CIDR>`
- Default route via `<HOST_LAN_GW>` on `eth1`
- No DHCP on `eth0`
- No DHCP on `eth1`
- Exactly one active Netplan YAML
- Docker ready for app workloads
- Docker is ready to publish intended app ports on `<C3_ETH1>` for LAN and Tailnet access once an app is deployed
- Docker data-root = `<DOCKER_DATA_ROOT>`
- Direct `IP:PORT` access is the default and sufficient access path for container 3 app workloads.
## Rules of thumb
- Keep the host minimal
- Keep AdGuard separate
- Do not run Docker on bare metal
- Only use `macvlan` where actual LAN visibility is required
- The host does not talk to `macvlan` containers via their LAN IP
- Document the port plan
- Host UFW protects host-bound services only
- LAN-visible container IPs are separate endpoints; expose only the ports you actually intend to publish there.
- No extra firewall inside the LXD containers in this design
- Disable cloud-init network config in the blueprint before cloning
- Build the blueprint first, then clone, then specialise
- Prefer bind mounts for app data under `<DOCKER_ROOT_DIR>/APPNAME`
- Use named volumes only when bind mounts are not workable or not worth debugging
- Keep Docker-managed internals under `<DOCKER_DATA_ROOT>`
- In Docker-in-LXD, check container user, ownership, and permissions first when a bind mount fails.
## Next steps
The foundation is now stable. Follow the “red thread” to
1. **the [Tailscale Runbook](../Optional/Tailscale%20Runbook.md "Tailscale Runbook")**: Establish secure remote access to your homelab (optional).
2. **the [Caddy Runbook](../Optional/Caddy%20Runbook.md "Caddy Runbook")**: Configure a reverse proxy and friendly local hostnames (optional).
3. **the [Docker Runbook](Docker%20Runbook.md "Docker Runbook")**: Learn how to manage your app workloads inside container 3.

Refer back to the [Homelab Guide](../Homelab%20Guide.md "Homelab Guide") at any time for the complete architecture overview and build sequence.