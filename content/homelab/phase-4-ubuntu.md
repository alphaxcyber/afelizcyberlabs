---
title: Phase 4 - Ubuntu Server
weight: 5
---

# Phase 4 — Ubuntu Server 🐧

## What is Ubuntu Server?

Ubuntu Server is a Linux distribution maintained by Canonical, designed specifically
for server workloads. Unlike the desktop version it has no graphical interface —
everything is managed through the terminal. This is intentional: no GUI means less
attack surface, lower memory usage, and it forces you to get comfortable with the
command line — an essential skill for any cybersecurity professional.

In this lab, Ubuntu Server serves as the primary Linux server VM — a stable,
well-documented platform for running services, practicing administration, and
eventually hosting security tools.

## Why Ubuntu 24.04 LTS Over Newer Versions

Ubuntu 24.04 LTS (Long Term Support) was chosen over newer interim releases
for several deliberate reasons:

- LTS releases receive security updates and patches for 5 years
- More mature package ecosystem — tools have had time to be tested and stabilized
- Industry standard — most enterprise environments and documentation target LTS releases
- Better compatibility with security tools like Wazuh, Elastic, and others in the stack

## VM Configuration

| Setting | Value |
|---|---|
| VM ID | 101 |
| Name | ubuntu-server-01 |
| BIOS | OVMF (UEFI) |
| Machine | q35 |
| CPU | 2 cores (host type) |
| RAM | 4GB (with ballooning) |
| Disk | 60GB SCSI, local-lvm, writeback cache, discard, iothread |
| NIC | VirtIO, bridge=vmbr1, firewall=1 |
| SCSI Controller | virtio-scsi-single |

Note: q35 is a modern machine type that supports PCIe and UEFI properly.
i440fx is the older alternative — q35 is the right choice for any VM
where you want modern hardware features.

## ISO Download

Ubuntu 24.04.4 LTS server ISO was downloaded directly into Proxmox local storage
using the URL download feature — no need to download to a workstation and upload
separately. This saves time and avoids the upload bottleneck.

## Installation Decisions

| Setting | Choice | Reason |
|---|---|---|
| Installation type | Ubuntu Server (full) | Not minimized — full package set available |
| Third party drivers | No | VM environment — no physical drivers needed |
| Network | Skipped | No DHCP on vmbr1 yet at install time |
| Proxy | None | Direct internet access |
| Storage | Entire disk + LVM, no encryption | Lab environment |
| SSH | OpenSSH Server installed | Remote access from XFlow |
| Ubuntu Pro | Skipped | Not needed for lab use |

## Disk Layout

| Partition | Size | Filesystem | Purpose |
|---|---|---|---|
| /boot/efi | 1.049GB | fat32 | UEFI boot partition |
| /boot | 2.000GB | ext4 | Kernel and boot files |
| / (ubuntu-lv) | 56.945GB | ext4 | Root filesystem (LVM) |
| swap | 3.8GB | swap | Virtual memory |

The LVM root volume was expanded from the Ubuntu installer default of ~28GB
to the full available disk space during installation. This is an important step —
the default leaves half the disk unused and requires manual expansion later.

## Post-Install Verification

After first boot, the following checks were run to confirm a healthy install:

```bash
uname -a
# Linux ubuntu-server-01 6.8.0-100-generic
# x86_64 GNU/Linux

df -h
# / (ubuntu-lv)  56G   6.4G   47G   12%  -- root filesystem healthy
# /boot          2.0G  103M   1.7G   6%  -- boot partition healthy
# /boot/efi      1.1G  6.2M   1.1G   1%  -- EFI partition healthy

free -h
# RAM:   3.8GB total, 402MB used
# Swap:  3.8GB total, 0 used

ip addr
# enp6s18: state DOWN (expected -- no DHCP on vmbr1 yet)
# MAC: bc:24:11:3d:00:53
```

## Static IP Configuration — Netplan

Ubuntu 24.04 uses Netplan for network configuration. Because the network was
skipped during installation, the /etc/netplan/ directory was empty — no config
file was created automatically. The file was created from scratch.

Why static IP for servers? DHCP addresses can change on reboot or lease renewal.
A server you SSH into needs a consistent, predictable address. Static IPs are
standard practice for any server in a production or lab environment.

```bash
# File: /etc/netplan/00-lab-network.yaml
# Permissions: 600 (root only -- required by Netplan)

network:
  version: 2
  ethernets:
    enp6s18:
      dhcp4: no
      addresses:
        - 172.16.0.10/24
      routes:
        - to: default
          via: 172.16.0.1
      nameservers:
        addresses:
          - 172.16.0.1
          - 8.8.8.8
```

Why chmod 600 on the Netplan file? Netplan throws a warning and may refuse to
apply configuration if the file is world-readable. Network config files can
contain sensitive information — restricting to root-only is correct security
practice regardless.

Applied with:
```bash
sudo chmod 600 /etc/netplan/00-lab-network.yaml
sudo netplan apply
```

Verified:
```bash
ip addr show enp6s18
# inet 172.16.0.10/24 -- static IP confirmed

ping 172.16.0.1    # OPNsense gateway -- 0% loss
ping 8.8.8.8       # Internet by IP -- 0% loss
ping google.com    # DNS + internet -- 0% loss
```

## SSH Configuration

OpenSSH Server was installed during the Ubuntu setup wizard. Ubuntu 24.04 defaults
to socket-based SSH activation — SSH only starts when a connection is requested
rather than running continuously. For a lab server this is unreliable, so it was
switched to always-on:

```bash
sudo systemctl enable ssh --now
```

The --now flag both enables the service (so it starts on boot) and starts it
immediately in a single command.

SSH access from XFlow_Machine was then tested through OPNsense:
```
ssh alphax@172.16.0.10
```
Result: Connected successfully via OPNsense WAN firewall rules.

## Network Interface Name — Why enp6s18?

Linux uses predictable network interface names based on hardware location.
enp6s18 breaks down as:

- en = ethernet
- p6 = PCIe bus 6
- s18 = slot 18

This naming is determined by where the virtual NIC appears in the VM's PCIe
bus layout. It will be consistent across reboots, unlike the old eth0/eth1
naming which could change depending on detection order.

## Snapshots Taken

| Snapshot | Description |
|---|---|
| fresh-install-clean | Ubuntu 24.04 fresh install, pre-network config |
| static-ip-network-working | Static IP 172.16.0.10, DNS working, internet confirmed |
| ssh-access-working | SSH from XFlow confirmed through OPNsense |

## VM Network Details

| Setting | Value |
|---|---|
| Interface | enp6s18 |
| Bridge | vmbr1 |
| IP | 172.16.0.10/24 (static) |
| Gateway | 172.16.0.1 (OPNsense LAN) |
| DNS | 172.16.0.1, 8.8.8.8 |
| MAC | bc:24:11:3d:00:53 |
| SSH | alphax@172.16.0.10 |

## What I Learned

- LTS vs interim Ubuntu releases and why LTS is the right choice for lab servers
- VirtIO paravirtualized drivers — why they outperform emulated hardware in VMs
- LVM (Logical Volume Manager) — how Ubuntu abstracts disk space into resizable
  logical volumes and why the default size should always be expanded during install
- UEFI vs legacy BIOS in VMs — q35 machine type and OVMF firmware
- Netplan YAML configuration — Ubuntu's network configuration layer
- YAML indentation rules — YAML is whitespace-sensitive, wrong indentation
  breaks the config silently
- Linux file permissions and why chmod 600 on network config files matters
- Predictable network interface naming (enp6s18) and how to read it
- Ubuntu 24.04 SSH socket activation vs always-on service
- Static IPs for servers — why DHCP is never appropriate for infrastructure
- The difference between systemctl enable (start on boot) and systemctl start
  (start now) and how --now combines both
