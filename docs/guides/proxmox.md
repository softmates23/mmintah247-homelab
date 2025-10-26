# Proxmox VE — Homelab Guide

> Opinionated install + baseline hardening for a single-node or small cluster.

## Prerequisites

- Hardware with virtualization support (Intel VT-x/AMD-V), 16GB+ RAM recommended
- 2 storage devices (fast NVMe for VM storage, SATA/NAS for backups)
- Static IP reserved in your router/DHCP

!!! tip
    Download the latest ISO with the web UI or `pveam status` after install to fetch templates quickly.

## 1) Install Proxmox VE

1. Flash ISO to USB (Ventoy/BalenaEtcher).
2. Boot and choose **Install Proxmox VE**.
3. Select target disk (NVMe recommended), set **Filesystem: ext4** or **ZFS (RAID1)** if you have 2 disks.
4. Set a **strong root password** and valid email.
5. Assign a **static IP**, gateway, DNS.

## 2) First login

- Browse to: `https://<proxmox-ip>:8006` → login as `root@pam`.
- Update:
  ```bash
  apt update && apt full-upgrade -y && pveupgrade
  ```
- Optional: enable non-subscription repo
  ```bash
  sed -i 's/^# deb/deb/' /etc/apt/sources.list.d/pve-enterprise.list || true
  echo 'deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription' | tee /etc/apt/sources.list.d/pve-no-subscription.list
  apt update
  ```

## 3) Networking (baseline)

- **vmbr0**: management + VM bridge
- **VLANs**: tag per network (e.g., `10` LAN, `20` IOT, `30` Servers)

```bash
cat /etc/network/interfaces
```
```ini
auto lo
iface lo inet loopback

auto enp3s0
iface enp3s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.30.10/24
    gateway 192.168.30.1
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
```

!!! note "VLAN aware"
    Enable **Bridge VLAN aware** in the UI if you trunk multiple VLANs to Proxmox.

## 4) Storage

- **Local-lvm**: thin pool for VM disks (NVMe)
- **Backups**: NFS/SMB to NAS, or Proxmox Backup Server

```bash
# Example: add NFS storage for backups
pvesm add nfs nas-backups --server 192.168.30.20 --path /mnt/pool/backups --content backup --maxfiles 7
```

## 5) Templates (cloud images)

```bash
pveam available | grep debian
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
# Create a template VM → then "Convert to template"
```

## 6) Backups & snapshots

- Schedule nightly **VM backups** to NAS storage.
- Keep 7–14 daily, 4 weekly.
- Use **stop mode** for small labs; **snapshot** if using qemu-guest-agent.

## 7) Secure baseline

- Create an admin user (realm `pve`) and add to `PVEAdmin`.
- Enable **TOTP** or **WebAuthn** 2FA.
- Lock down firewall:
  - Datacenter Firewall → **enable**
  - Node Firewall → **enable**
  - Accept inbound 22/tcp and 8006/tcp from admin subnet only
- Install guest agent on VMs:
  ```bash
  apt install -y qemu-guest-agent && systemctl enable --now qemu-guest-agent
  ```

## 8) Quality-of-life

- **Tags** for VMs: `prod`, `lab`, `infra`
- **Notes**: record IPs, roles, and backup policy per VM
- **Hooks**: pre/post backup scripts for app quiesce

## Troubleshooting

=== "VM has no network"
    - Check VLAN ID on NIC + switch port
    - Verify `bridge-vlan-aware yes`

=== "Backups failing"
    - NAS permissions for `backup` content
    - Check `journalctl -u pvedaemon`

## Appendix: VM Sizing Cheatsheet

| Role | vCPU | RAM | Disk |
|---|---:|---:|---:|
| Pi-hole/DNS | 1 | 512MB | 4–8GB |
| Home Assistant | 2 | 2–4GB | 32GB |
| k3s server | 2–4 | 4–8GB | 40GB |
| Grafana/Prometheus | 2 | 4–8GB | 40–80GB |
