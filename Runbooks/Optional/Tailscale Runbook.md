# Tailscale Runbook
> Placeholders: [Homelab Guide](../Homelab%20Guide.md#placeholders "Homelab Guide").
## Scope
Install and configure Tailscale inside container 2 to provide secure remote access to the homelab and to act as a subnet router for the internal bridge and LAN.
## Prerequisites
- The [Foundation Runbook](../Essential/Foundation%20Runbook.md "Foundation Runbook") is complete up to phase 3 (DNS container running).
- A Tailscale account is available.
- Internet access is available on container 2.
## Target state
- Tailscale is installed and authenticated inside container 2.
- Container 2 advertises routes for `<LXDBR0_SUBNET>` and `<LAN_CIDR>`.
- The homelab is reachable from other Tailnet clients via the Tailscale IP of container 2.
## 1. Install Tailscale
Run inside container 2:
```sh
curl -fsSL https://tailscale.com/install.sh | sh
```
## 2. Enable IP forwarding
Tailscale requires IP forwarding to act as a subnet router.

Run inside container 2:
```sh
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```
## 3. Authenticate and start
Run inside container 2:
```sh
sudo tailscale up --accept-dns=false --advertise-routes=<LXDBR0_SUBNET>,<LAN_CIDR> --accept-routes
```
Expected behaviour:
- Tailscale prints an authentication URL.
- Open that URL from another device in a browser.
- Sign in to your Tailscale account and authorise this node.
- Return to the container and confirm the node appears in `tailscale status`.
## 4. Verify and approve routes
1. Log in to the Tailscale admin console.
2. Find the node for container 2.
3. Under “Edit route settings”, approve the advertised routes for `<LXDBR0_SUBNET>` and `<LAN_CIDR>`.
## 5. Configure return routes (Required for other containers)
If you want other containers (like container 3) to be reachable via Tailscale, they must know how to route traffic back to the Tailnet.

Run inside container 3 (and any other container that needs Tailnet access):
```sh
sudo ip route add 100.64.0.0/10 via <C2_ETH0> dev eth0
```
To make this persistent, add it to the `eth0` routes in the container’s Netplan configuration.
## 6. Validation
Run inside container 2:
```sh
tailscale status
tailscale ip
ip -o route get <FALLBACK_NAMESERVER>
```
Verify from a remote Tailnet client:
- Ping the Tailscale IP of container 2.
- Ping `<C2_ETH0>` (internal bridge IP) or `<C3_ETH1>` (Docker LAN IP) to verify subnet routing.
## Notes
- Tailscale uses `eth1` as the effective uplink for traffic reaching the LAN.
- Ensure the Tailscale dashboard shows the node as active and the routes as approved.