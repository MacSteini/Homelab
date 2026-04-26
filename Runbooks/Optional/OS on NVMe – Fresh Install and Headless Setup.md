# OS on NVMe – Fresh Install and Headless Setup
> Placeholders: [Homelab Guide](../Homelab%20Guide.md#placeholders "Homelab Guide").
## Scope
Prepare a fresh Ubuntu Server install for headless use, boot the Raspberry Pi from NVMe, create the primary admin user, and finish with a host that is reachable over SSH for the [Foundation Runbook](../Essential/Foundation%20Runbook.md "Foundation Runbook").
## Prerequisites
- Raspberry Pi 4 or 5
- NVMe SSD and a compatible adapter/HAT
- MicroSD card (optional, only if bootloader update is required)
- Another machine for flashing the image and SSH access
## 1. Prepare the installation medium
1. Download the latest Ubuntu Server LTS image for ARM64.
2. Use Raspberry Pi Imager or `dd` to flash the image to your NVMe SSD.
3. Before ejecting, configure the following “OS Customisation” settings:
	- **Hostname:** homelab
	- **Username:** `<SSH_ADMIN_USER>`
	- **Password:** a strong password
	- **SSH:** Enable SSH, use password authentication or provide `<SSH_PUBKEY_PATH>`
	- **Wireless LAN:** optional (wired ethernet is preferred)
	- **Locale:** set your timezone and keyboard layout
## 2. Boot the Raspberry Pi
1. Connect the NVMe SSD to the Raspberry Pi.
2. Ensure no other bootable media (like a micro SD card) is inserted unless you are deliberately using it to update the bootloader.
3. Power on the Raspberry Pi.
4. Wait for the initial boot process and cloud-init to complete.
## 3. First SSH access
From your admin machine, run:
```sh
ssh <SSH_ADMIN_USER>@<HOST_LAN_IP>
```
If `<HOST_LAN_IP>` is not yet known, check your router’s DHCP lease table for a device named `homelab`.
## 4. Verify system state
Run on the host:
```sh
hostnamectl
id
findmnt /
lsblk
```
Expected state:
- `id` shows `<SSH_ADMIN_USER>`
- `/` is on the installed system disk
- The NVMe device is visible in `lsblk`
- The system is booted from the intended fresh install, not from an older fallback medium
## 5. Confirm the host is ready for homelab setup
Run on the host:
```sh
sudo -v
sudo apt update
```
At this stage, do not replace the network layout yet.
## 6. Continue with the foundation
Continue with the [Foundation Runbook](../Essential/Foundation%20Runbook.md "Foundation Runbook") to build the container infrastructure.

The next runbook will:
- Normalise Netplan for the host
- Install and initialise LXD
- Configure the firewall
- Build containers 1–3
## Validation
From the admin machine:
```sh
ssh <SSH_ADMIN_USER>@<HOST_LAN_IP>
```
Run on the host:
```sh
hostnamectl
id <SSH_ADMIN_USER>
findmnt /
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS
sudo -v
sudo apt update
```
## Notes
- Keep this runbook focused on fresh install and first headless access only.
- Do not duplicate the later host build from the [Foundation Runbook](../Essential/Foundation%20Runbook.md "Foundation Runbook").
- Do not use this runbook for adding or preparing NVMe storage on an already running Ubuntu system.