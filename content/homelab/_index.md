---
title: Homelab Build Series
weight: 2
---

# Homelab Build Series 🖥️

![A teal-lit data center server corridor](datacenterserverroom.jpg)

Complete build documentation of my Proxmox-based cybersecurity home lab — built from scratch on enterprise hardware, documented step by step.

## Hardware
- **Server:** HP Z440 (Xeon E5-2690 v4, 64GB ECC RAM)
- **Hypervisor:** Proxmox VE 8.4
- **Firewall:** OPNsense 25.1
- **NICs:** Intel eno1 (management) + Fenvi Dual Port I226-V (WAN)

## Lab Network

| IP | Device |
|---|---|
| 172.16.0.1 | OPNsense LAN gateway |
| 172.16.0.10 | Ubuntu Server |
| 172.16.0.20 | Kali Linux |
| 172.16.0.30 | Windows Server 2022 |
| 172.16.0.40 | Security Onion (Phase 2) |
| 172.16.0.50 | Wazuh SIEM (Phase 2) |

## Build Phases

Navigate the phases using the sidebar to follow the full build from bare metal to fully operational security lab.