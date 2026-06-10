---
title: Phase 6 - Windows Server 2022
weight: 7
---

# Phase 6 — Windows Server 2022 🪟

## What is Windows Server 2022?

Windows Server 2022 is Microsoft's current server operating system, used in the
vast majority of enterprise environments worldwide. It provides Active Directory
Domain Services (AD DS), Group Policy, DNS, DHCP, file sharing, and a wide range
of enterprise services that form the backbone of corporate IT infrastructure.

In this lab, Windows Server 2022 serves two purposes. First, it provides a
realistic enterprise target environment — understanding how Windows Server works
is essential for both attacking and defending corporate networks. Second, it will
be promoted to a Domain Controller running Active Directory, which enables a full
range of AD attack and defense scenarios that are central to the OSCP exam and
real-world SOC work.

## Why Windows Server in a Cybersecurity Lab

The majority of enterprise environments run Windows infrastructure. Active Directory
is present in nearly every mid-to-large organization. Understanding AD — how it
works, how it is attacked, and how to detect those attacks — is one of the most
valuable skills a cybersecurity professional can have.

Attacks like Kerberoasting, Pass-the-Hash, DCSync, and BloodHound enumeration
are staples of penetration testing certifications and real-world red team engagements.
You cannot practice defending against these without a real AD environment to work in.

## VM Configuration

| Setting | Value |
|---|---|
| VM ID | 103 |
| Name | windows-server-2022 |
| BIOS | SeaBIOS |
| CPU | 2 cores (host type) |
| RAM | 8GB |
| Disk | 80GB, local-lvm |
| NIC | VirtIO, bridge=vmbr1 |
| OS | Windows Server 2022 Standard Evaluation (Desktop Experience) |

## VirtIO Drivers — A Critical Requirement

Windows cannot see VirtIO virtual hardware during installation without drivers.
VirtIO is a paravirtualized device standard — it is faster than emulated hardware
but requires guest drivers to be installed.

Without the VirtIO storage driver loaded during setup, the Windows installer
shows no available disks and installation cannot proceed.

### Storage Driver — Loaded During Installation

The virtio-win ISO was attached as a second CD drive in Proxmox before starting
the Windows installer. During the "Where do you want to install Windows?" screen:

1. Click Load driver
2. Browse to the virtio-win CD
3. Navigate to viostor\2k22\amd64
4. Select and install the VirtIO SCSI storage driver
5. The 80GB disk appears and installation can proceed

### Network and Remaining Drivers — Installed Post-Boot

After Windows finished installing and booted for the first time, the remaining
VirtIO drivers were installed using the guest tools installer:

```
virtio-win-guest-tools.exe
```

This installs all remaining drivers in one pass:
- Network adapter (NetKVM)
- Memory balloon driver
- QEMU guest agent
- PCI serial port driver

The QEMU guest agent is particularly useful — it allows Proxmox to communicate
with the VM for clean shutdowns, IP address reporting, and snapshot consistency.

## Static IP Configuration

Static IP was set via the Windows GUI:

```
Control Panel -- Network and Internet -- Network Connections
(or run: ncpa.cpl)

Right-click Ethernet adapter -- Properties
Internet Protocol Version 4 (TCP/IPv4) -- Properties

IP address:      172.16.0.30
Subnet mask:     255.255.255.0
Default gateway: 172.16.0.1
DNS server:      172.16.0.1
Alternate DNS:   8.8.8.8
```

## RDP Configuration

Remote Desktop Protocol was enabled for remote access from XFlow_Machine:

```
Settings -- System -- Remote Desktop -- Enable Remote Desktop: On
```

ICMP (ping) was enabled via PowerShell since Windows Firewall blocks it by default:

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4" `
protocol=icmpv4:8,any dir=in action=allow
```

RDP access from XFlow was tested:
```
mstsc /v:172.16.0.30
Username: Administrator
```
Result: Connected successfully via OPNsense WAN firewall rules.

## VM Network Details

| Setting | Value |
|---|---|
| Bridge | vmbr1 |
| IP | 172.16.0.30/24 (static) |
| Gateway | 172.16.0.1 (OPNsense LAN) |
| DNS | 172.16.0.1, 8.8.8.8 |
| RDP | mstsc /v:172.16.0.30 |
| Username | Administrator |

## Snapshots Taken

| Snapshot | Description |
|---|---|
| fresh-install-rdp-working | Windows Server fresh install, VirtIO drivers, RDP confirmed |

## Planned: Active Directory Domain Controller

The next major step for this VM is promoting it to a Domain Controller and
standing up a full Active Directory environment. This will enable the full
AD attack and defense curriculum.

Planned configuration:

| Setting | Value |
|---|---|
| Domain name | lab.local |
| Domain Controller | VM 103 (this machine) |
| Forest functional level | Windows Server 2022 |

Once AD is configured, the lab curriculum will include:

- **Enumeration** — BloodHound, ldapdomaindump, enum4linux
- **Kerberoasting** — extracting and cracking service account tickets
- **AS-REP Roasting** — attacking accounts without pre-authentication
- **Pass-the-Hash** — lateral movement using NTLM hashes
- **DCSync** — replicating domain credentials from a compromised account
- **Detection** — Security Onion and Wazuh alerts for each attack technique

## What I Learned

- Why Windows Server is essential in a cybersecurity lab — AD is everywhere in
  enterprise environments
- VirtIO paravirtualized drivers — why Windows cannot see virtual disks without
  them and how to load them during installation
- The difference between loading a driver during Windows setup vs installing
  post-boot with the guest tools installer
- QEMU guest agent — what it does and why it matters for VM management
- Windows Firewall defaults — ICMP is blocked by default and must be explicitly
  allowed
- RDP configuration and the mstsc command for remote desktop from Windows
- ncpa.cpl as a shortcut to network adapter settings
- Why 8GB RAM was allocated vs 4GB for Linux VMs — Windows Server has higher
  baseline memory requirements
- The relationship between Active Directory and cybersecurity — why AD attacks
  are central to penetration testing certifications and real-world engagements
