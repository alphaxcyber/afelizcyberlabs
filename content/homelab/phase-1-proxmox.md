---
title: Phase 1 - Proxmox Installation
weight: 2
---

# Phase 1 — Proxmox Installation 🖥️

## What is Proxmox?

Proxmox VE (Virtual Environment) is a bare metal hypervisor — meaning it runs directly on the hardware with no underlying OS beneath it. It lets you run multiple virtual machines simultaneously on a single physical server, each isolated from one another. Think of it as the foundation everything else in the lab sits on top of.

## Hardware Used

| Component | Spec |
|---|---|
| CPU | Xeon E5-2690 v4 (14 cores / 28 threads) |
| RAM | 64GB ECC |
| GPU | AMD RX6600XT 8GB (future AI/ML use) |
| Storage (OS) | 500GB SSD |
| Storage (Expansion) | 1TB SSD |
| NIC 1 | Onboard Intel eno1 |
| NIC 2 | Fenvi Dual Port Intel I226-V PCIe 2.5G |

## Installation Steps

### 1. Download Proxmox VE 8.4 ISO
Chose 8.4 over 9.2 for stability and broader community support. Newer isn't always better in a lab environment — maturity means more documentation and more solved problems.

### 2. Create Bootable USB
Used Rufus in **DD image mode** — not ISO mode. This matters because Proxmox uses a hybrid ISO that requires DD mode to boot correctly on UEFI systems.

### 3. BIOS Configuration
| Setting | Value |
|---|---|
| Boot Mode | UEFI |
| Secure Boot | Disabled |
| VT-x | Enabled |
| VT-d | Enabled |

VT-x enables CPU virtualization. VT-d enables IOMMU — required later for PCIe passthrough (GPU, NIC).

### 4. Installation
- Target disk: 500GB SSD
- Filesystem: ext4 (simple, well supported)
- Static IP: 192.168.0.10/24
- Gateway: 192.168.0.1

### 5. Post-Installation Configuration

**Disable enterprise repositories** (requires paid subscription):
```bash
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list
```

**Add free community repository, then update:**
```bash
apt update && apt full-upgrade -y
```

**Remove subscription nag screen:**
```bash
sed -i.bak "s/NotSubscribed/Active/" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

> ⚠️ **Note:** The nag fix needs to be reapplied after every Proxmox update, as updates overwrite proxmoxlib.js.

## Current Status

| Setting | Value |
|---|---|
| Version | Proxmox VE 8.4.19 |
| Web UI | https://192.168.0.10:8006 |
| Username | root |
| Repos | Community (free) |
| Status | Fully updated ✅ |

## What I Learned

- Proxmox installation writes its own UEFI bootloader as the primary entry, which pushed my existing Windows Boot Manager down — making Windows unbootable. Always verify boot order after installing a bare metal hypervisor alongside an existing OS.
- The difference between VT-x (CPU virtualization) and VT-d (IOMMU/passthrough) — both need to be enabled for a full-featured lab.
- Why enterprise repos are disabled for home use — Proxmox is free but defaults to a subscription-gated update channel.
