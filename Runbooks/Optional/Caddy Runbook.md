# Caddy Runbook
> Placeholders: [Homelab Guide](../Homelab%20Guide.md#placeholders "Homelab Guide").
## Scope
Deploy Caddy as an optional reverse proxy inside container 3 so selected Docker apps are reachable via friendly local domain names (e. g., `service.<LOCAL_DOMAIN>`) instead of `IP:PORT`.
## Prerequisites
- The [Foundation Runbook](../Essential/Foundation%20Runbook.md "Foundation Runbook") is complete.
- Container 3 is running and Docker is installed.
- DNS container (container 2) is operational.
## Target state
- Caddy runs in Docker inside container 3.
- Caddy listens on `80/tcp` and `443/tcp` on `<C3_ETH1>`.
- Selected apps are reachable via `https://app.<LOCAL_DOMAIN>`.
- AdGuard Home rewrites `<LOCAL_DOMAIN>` records to `<C3_ETH1>`.
- HTTP and HTTPS both work; HTTP→HTTPS redirects remain disabled by default.
## 1. Prepare directories
Run inside container 3:
```sh
sudo mkdir -p <DOCKER_STACKS_DIR>/caddy
sudo mkdir -p <DOCKER_ROOT_DIR>/caddy/data
sudo mkdir -p <DOCKER_ROOT_DIR>/caddy/config
```
## 2. Create the compose file
Create `<DOCKER_STACKS_DIR>/caddy/compose.yml`:
```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - <DOCKER_STACKS_DIR>/caddy/Caddyfile:/etc/caddy/Caddyfile
      - <DOCKER_ROOT_DIR>/caddy/data:/data
      - <DOCKER_ROOT_DIR>/caddy/config:/config
    networks:
      - <DOCKER_NETWORK>

networks:
  <DOCKER_NETWORK>:
    external: true
```
## 3. Create the Caddyfile
Create `<DOCKER_STACKS_DIR>/caddy/Caddyfile`.

Minimal example for AdGuard Home and an example app:
```caddy
# AdGuard Home
ag.<LOCAL_DOMAIN> {
	reverse_proxy <C2_ETH1>:3000
}

# Example App
app.<LOCAL_DOMAIN> {
	reverse_proxy appname:<DOCKER_APP_PORT>
}

# Optional: Global options for local development
# {
# 	# Disable administrative API
# 	admin off
# }
```
## 4. Start Caddy
Run inside container 3:
```sh
cd <DOCKER_STACKS_DIR>/caddy && sudo docker compose up -d
```
## 5. Configure DNS rewrites
In the AdGuard Home UI (container 2):
1. Navigate to **Filters** → **DNS rewrites**.
2. Add a rewrite: `*.<LOCAL_DOMAIN>` → `<C3_ETH1>`.
## 6. Validation
Run from another LAN client:
```sh
curl -I http://ag.<LOCAL_DOMAIN>
curl -kI https://ag.<LOCAL_DOMAIN>
```
## Notes
- Caddy publishes `80` and `443` on the LAN via `<C3_ETH1>`.
- Direct `IP:PORT` access remains functional for troubleshooting.
- To trust the Caddy CA on your devices (optional), export the root certificate from the container’s `/data/caddy/pki/authorities/local/root.crt`.