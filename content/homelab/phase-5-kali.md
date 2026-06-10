---
title: Phase 5 - Kali Linux
weight: 6
---

# Phase 5 — Kali Linux 🐉

## What is Kali Linux?

Kali Linux is a Debian-based Linux distribution maintained by Offensive Security,
designed specifically for penetration testing, digital forensics, and security
research. It comes pre-loaded with hundreds of security tools — network scanners,
exploit frameworks, password crackers, wireless analysis tools, and more.

In this lab, Kali serves as the primary attack platform. It is the machine used
to simulate adversarial activity against other lab VMs — running scans, testing
firewall rules, practicing exploitation techniques, and learning the offensive
side of cybersecurity that makes defensive work meaningful.

Understanding how attackers think and operate is fundamental to becoming an
effective defender. You cannot build good detection rules for what you have
never seen an attacker do.

## Why Kali Linux Specifically

Several penetration testing distributions exist — Parrot OS, BlackArch, and others.
Kali was chosen for several reasons:

- Industry standard — the overwhelming majority of penetration testing courses,
  certifications (eJPT, OSCP), and writeups assume Kali
- Best tool coverage out of the box — no need to manually install most tools
- Largest community — more documentation, more troubleshooting resources
- Official Offensive Security support — tools are maintained and updated regularly
- Required for OSCP — the exam environment is built around Kali

## VM Configuration

| Setting | Value |
|---|---|
| VM ID | 102 |
| Name | kali-linux |
| BIOS | SeaBIOS |
| CPU | 2 cores (host type) |
| RAM | 4GB |
| Disk | 60GB, local-lvm |
| NIC | VirtIO, bridge=vmbr1 |
| OS | Kali Linux 2026.1 |

## Installation

Kali Linux 2026.1 ISO was downloaded into Proxmox local storage and installed
as a standard VM. The graphical installer was used with default options, selecting
the full Kali desktop environment for the complete tool suite.

## Static IP Configuration

Kali Linux uses the traditional Debian /etc/network/interfaces file for network
configuration — not Netplan like Ubuntu. This is an important distinction:
different Linux distributions handle network configuration differently, and
recognizing which system a distribution uses is a core Linux administration skill.

```bash
# File: /etc/network/interfaces

auto eth0
iface eth0 inet static
    address 172.16.0.20
    netmask 255.255.255.0
    gateway 172.16.0.1
    dns-nameservers 172.16.0.1 8.8.8.8
```

Applied by restarting the networking service:
```bash
sudo systemctl restart networking
```

Verified:
```bash
ip addr show eth0
# inet 172.16.0.20/24 -- static IP confirmed

ping 172.16.0.1    # OPNsense gateway -- 0% loss
ping 8.8.8.8       # Internet by IP -- 0% loss
ping google.com    # DNS + internet -- 0% loss
```

## SSH Configuration

OpenSSH Server was installed and enabled for remote access from XFlow_Machine:

```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh --now
```

SSH access from XFlow was tested through OPNsense:
```
ssh alphax@172.16.0.20
```
Result: Connected successfully via OPNsense WAN firewall rules.

## Network Interface Name — Why eth0?

Kali uses eth0 rather than the predictable naming format (like enp6s18 on Ubuntu)
because Kali disables predictable interface naming by default. This is intentional
for a penetration testing distribution — many tools and scripts in the security
field hardcode eth0, wlan0, and similar traditional names. Using predictable names
would break compatibility with a large body of existing tooling.

This is a good example of how the same underlying Linux kernel behaves differently
across distributions based on deliberate design choices for their target use case.

## Key Tools Available in Kali

Kali ships with hundreds of tools organized by category. The ones most relevant
to this lab's curriculum:

| Category | Tools |
|---|---|
| Network scanning | Nmap, Masscan, Netdiscover |
| Vulnerability scanning | OpenVAS, Nikto, WPScan |
| Exploitation | Metasploit Framework, SearchSploit |
| Password attacks | Hydra, John the Ripper, Hashcat, CrackMapExec |
| Web application | Burp Suite, SQLmap, Dirb, Gobuster |
| Wireless | Aircrack-ng, Kismet, Wifite |
| Forensics | Autopsy, Volatility, Binwalk |
| OSINT | theHarvester, Maltego, Recon-ng |
| Post-exploitation | Empire, Covenant, Mimikatz |

## OPNsense DHCP Reservation

A static DHCP reservation was added in OPNsense to ensure Kali always receives
the correct IP if DHCP is ever used:

| MAC | IP | Device |
|---|---|---|
| bc:24:11:63:8e:bd | 172.16.0.20 | Kali Linux |

Note: Even with a static IP configured in /etc/network/interfaces, having the
DHCP reservation ensures consistency if the static config is ever removed or
the interface is reset.

## VM Network Details

| Setting | Value |
|---|---|
| Interface | eth0 |
| Bridge | vmbr1 |
| IP | 172.16.0.20/24 (static) |
| Gateway | 172.16.0.1 (OPNsense LAN) |
| DNS | 172.16.0.1, 8.8.8.8 |
| MAC | bc:24:11:63:8e:bd |
| SSH | alphax@172.16.0.20 |

## Snapshots Taken

| Snapshot | Description |
|---|---|
| fresh-install-ssh-working | Kali fresh install, static IP, SSH from XFlow confirmed |

## Planned Use in the Lab

As the lab grows, Kali will be used for:

- **Network reconnaissance** — scanning other lab VMs to understand their exposed
  attack surface and verify firewall rules are working as intended
- **Exploitation practice** — attacking Metasploitable and other intentionally
  vulnerable VMs in a controlled environment
- **CTF challenges** — working through VulnHub and HackMyVM machines
- **OSCP preparation** — building the methodology and muscle memory for the exam
- **Detection validation** — generating attack traffic that Security Onion and Wazuh
  should detect, then verifying alerts fire correctly

## What I Learned

- Kali Linux's purpose and why it is the industry standard for penetration testing
- Debian /etc/network/interfaces vs Ubuntu Netplan — different distros, different
  network configuration systems
- Why Kali disables predictable interface naming (eth0 instead of enp6s18)
- The relationship between offensive and defensive security — you need to understand
  attacks to build effective defenses
- How to install and enable SSH on a fresh Kali install
- OPNsense DHCP reservations — binding a MAC address to a specific IP in the
  firewall's DHCP server for consistent addressing
- The importance of having a dedicated attack VM isolated from other lab systems
  on the same internal network segment
