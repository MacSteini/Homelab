# Docker Runbook
> Placeholders: [Homelab Guide](../Homelab%20Guide.md#placeholders "Homelab Guide").
## Scope
Manage and operate Docker workloads inside container 3. This runbook covers application deployment, troubleshooting, and the complete removal of the Docker stack if required.

The Docker engine itself is provisioned during the [Foundation Runbook](Foundation%20Runbook.md "Foundation Runbook") build.
## Prerequisites
`<C3_NAME>` running, Docker installed, `data-root` = `<DOCKER_DATA_ROOT>`, `<DOCKER_NETWORK>` network created.
## Deploy a new app
### 1. Create directories
Run inside `<C3_NAME>`:
```sh
sudo mkdir -p <DOCKER_STACKS_DIR>/APPNAME && sudo mkdir -p <DOCKER_ROOT_DIR>/APPNAME
```
### 2. Write compose file
Create `<DOCKER_STACKS_DIR>/APPNAME/compose.yml`.

Minimal bind-mount example:
```yaml
services:
  appname:
    image: IMAGE:TAG
    container_name: appname
    restart: unless-stopped
    ports:
      - "<DOCKER_APP_PORT>:<DOCKER_APP_PORT>"
    volumes:
      - <DOCKER_ROOT_DIR>/APPNAME:/app/data
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:<DOCKER_APP_PORT>/ >/dev/null 2>&1 || exit 1"]
      interval: 60s
      timeout: 10s
      retries: 5
      start_period: 30s
    networks:
      - <DOCKER_NETWORK>

networks:
  <DOCKER_NETWORK>:
    external: true
```
### 3. Start
```sh
cd <DOCKER_STACKS_DIR>/APPNAME && sudo docker compose up -d && sudo docker compose ps
```
### 4. Verify
```sh
cd <DOCKER_STACKS_DIR>/APPNAME && sudo docker compose ps && sudo docker compose logs --tail=50 && curl http://127.0.0.1:<DOCKER_APP_PORT>
```
If the app should be LAN-reachable from outside container 3, verify it against `<C3_ETH1>:<DOCKER_APP_PORT>` from another LAN client. For remote Tailnet use, prefer direct `IP:PORT` or the app’s Tailnet-reachable address. Do not use the host itself for this check when `eth1` is a `macvlan` interface.
## Compose conventions
### Bind mount (default)
```yaml
volumes:
  - <DOCKER_ROOT_DIR>/APPNAME:/app/data
```
### Named volume (exception only)
```yaml
services:
  appname:
    volumes:
      - APPNAME_data:/app/data

volumes:
  APPNAME_data:
    name: APPNAME
```
> `name: APPNAME` sets the Docker-managed volume name explicitly. Without it, Docker generates `stackdirname_APPNAME_data` – unpredictable and hard to inspect.

> Use named volumes only when bind mounts fail due to UID/GID or Docker-in-LXD permission issues. Named volumes are stored under `<DOCKER_DATA_ROOT>/volumes/` via `data-root`.
### Shared network
Every compose file must declare the external `<DOCKER_NETWORK>` network:
```yaml
networks:
  <DOCKER_NETWORK>:
    external: true
```
### Exposure classes
Use one of these three patterns deliberately:

Class|Use|Binding rule|Validation rule
-|-|-|-
User-facing apps|Apps reached directly on the LAN or via Tailnet|Publish on `<C3_ETH1>` with `PORT:PORT`|Validate by direct `IP:PORT`
Internal-only helpers|Infrastructure helpers that are not user-facing|Keep them off published LAN ports|Validate from inside `<C3_NAME>`

### Port binding

Bind|Meaning
-|-
`PORT:PORT`|Reachable on all container interfaces
`127.0.0.1:PORT:PORT`|Container-local only

### Standard healthcheck pattern
```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:PORT/ >/dev/null 2>&1 || exit 1"]
  interval: 60s
  timeout: 10s
  retries: 5
  start_period: 30s
```
## Operations
### Logs
```sh
sudo docker compose -f <DOCKER_STACKS_DIR>/APPNAME/compose.yml logs -f
```
### Restart
```sh
sudo docker compose -f <DOCKER_STACKS_DIR>/APPNAME/compose.yml restart
```
### Update
```sh
cd <DOCKER_STACKS_DIR>/APPNAME && sudo docker compose pull && sudo docker compose up -d && sudo docker image prune -f
```
### Shell access
```sh
sudo docker exec -it CONTAINER_NAME bash
```
### Inspect mounts
```sh
sudo docker inspect CONTAINER_NAME --format '{{json .Mounts}}' | python3 -m json.tool && sudo docker inspect CONTAINER_NAME --format '{{.Config.User}}'
```
### Named volume location
```sh
sudo docker volume inspect VOLUME_NAME
# HostPath will be under <DOCKER_DATA_ROOT>/volumes/
```
## Troubleshooting: Docker-in-LXD
### Bind mount fails (permission denied)
```sh
ls -ld <DOCKER_ROOT_DIR>/APPNAME && namei -l <DOCKER_ROOT_DIR>/APPNAME && sudo docker inspect CONTAINER_NAME --format '{{.Config.User}}'
```
> Check ownership vs. the UID the container process runs as. `chown` the bind-mount directory to match, or fall back to a named volume.
### Container fails to start (OCI/runtime error)
Verify LXD nesting config on the host:
```sh
sudo lxc config show <C3_NAME> | grep -E 'security\.(nesting|syscalls)'
```
Expected:
```text
security.nesting: "true"
security.syscalls.intercept.mknod: "true"
security.syscalls.intercept.setxattr: "true"
```
If missing, re-apply “LXD nesting prerequisites for Docker” from the [Foundation Runbook](Foundation%20Runbook.md#lxd-nesting-prerequisites-for-docker "Foundation Runbook").
## Nuke
Remove Docker from `<C3_NAME>` completely when you want to rebuild the Docker layer from scratch.

This section is intentionally destructive. It removes Docker Engine, CLI state, Compose state, images, containers, networks, volumes, runtime data, and the Docker configuration on container 3.
> Warning: all Docker-managed data under `<DOCKER_DATA_ROOT>` and the default runtime paths is deleted permanently.
### Stop Docker and related services
```sh
sudo systemctl stop docker docker.socket containerd && sudo systemctl disable docker docker.socket containerd
```
### Remove packages
```sh
sudo apt purge -y docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras docker.io containerd containerd.io runc && sudo apt autoremove -y --purge
```
### Delete runtime data and config
```sh
sudo rm -rf <DOCKER_DATA_ROOT> /var/lib/docker /var/lib/containerd /etc/docker /etc/containerd /run/docker /run/containerd /var/run/docker.sock
```
If your bind-mounted application data under `<DOCKER_ROOT_DIR>` should also be removed, do that deliberately and per app. Do not delete `<DOCKER_ROOT_DIR>` blindly unless a full app-data wipe is really intended.
### Remove repository configuration
```sh
grep -R "download.docker.com" /etc/apt/sources.list /etc/apt/sources.list.d 2>/dev/null || true
ls -l /etc/apt/keyrings /usr/share/keyrings 2>/dev/null | grep -i docker || true
sudo rm -f /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.sources /etc/apt/keyrings/docker.gpg /etc/apt/keyrings/docker.asc /usr/share/keyrings/docker-archive-keyring.gpg && sudo apt update
```
### Remove user-level Docker state
```sh
rm -rf ~/.docker && sudo rm -rf /root/.docker
```
### Remove Docker group
```sh
sudo delgroup docker || true
```
### Verify removal
```sh
command -v docker || echo "docker binary: gone" && systemctl list-unit-files | grep docker || echo "no docker units" && ls <DOCKER_DATA_ROOT> 2>/dev/null || echo "docker data-root: gone"
```
Expected result:
- Docker binary removed
- No Docker systemd units left
- `<DOCKER_DATA_ROOT>` no longer present
## Validation
```sh
sudo docker ps && sudo docker network inspect <DOCKER_NETWORK> && sudo docker info | grep "Docker Root Dir" && sudo docker system df
```