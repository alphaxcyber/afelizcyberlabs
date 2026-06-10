---
title: Phase 3 - OPNsense Firewall
weight: 4
---

# Phase 3 — OPNsense Firewall 🔥

## What is OPNsense?

OPNsense is an open-source firewall and router platform built on FreeBSD — not Linux,
an important distinction. It handles all traffic entering and leaving the lab network:
NAT, firewall rules, DHCP, DNS, and routing. Think of it as the security perimeter
between the home network and the lab.

## VM Configuration

| Setting | Value |
|---|---|
| VM ID | 100 |
| Name | OPNsense-Firewall |
| BIOS | SeaBIOS |
| CPU | 2 cores (host type) |
| RAM | 2GB |
| Disk | 20GB (ZFS stripe) |
| net0 (WAN) | VirtIO, MAC bc:24:11:8a:88:5f, bridge=vmbr0 |
| net1 (LAN) | VirtIO, MAC bc:24:11:90:ca:b7, bridge=vmbr1 |

## Installation

- Downloaded OPNsense 25.1 DVD ISO directly into Proxmox local storage
- Installed using ZFS stripe filesystem
- Completed setup wizard

## Interface Configuration

OPNsense presented two virtual interfaces: vtnet0 and vtnet1. They were initially
assigned backwards. The correct assignment was verified by MAC address using
ifconfig vtnet1 in the OPNsense shell, then corrected via console Option 1.

| Interface | vtnet | Bridge | Role | IP |
|---|---|---|---|---|
| WAN | vtnet0 | vmbr0 | Internet uplink | 192.168.0.207 (DHCP) |
| LAN | vtnet1 | vmbr1 | Lab gateway | 172.16.0.1/24 |

Lesson: Always verify interface assignments by MAC address, not by name alone.
Interface names (vtnet0, vtnet1) do not always map to what you expect.

## OPNsense Services

| Service | Configuration |
|---|---|
| DHCPv4 | 172.16.0.100 — 172.16.0.200 |
| DNS Resolver | Unbound (enabled) |
| DNSSEC | Enabled |
| NTP | 0-3.opnsense.pool.ntp.org |
| NAT | Enabled (lab VMs share OPNsense WAN IP) |
| Web GUI | http://172.16.0.1 |

## Critical Configuration Settings

### 1. Block Private Networks on WAN — UNCHECKED

By default OPNsense blocks RFC1918 private addresses on the WAN interface
(10.x, 172.16.x, 192.168.x). This is correct for a real internet-facing WAN —
you would never expect legitimate traffic from private IPs on a public interface.

However, this lab's WAN is on 192.168.0.x — a private subnet. If this setting
stays enabled, OPNsense blocks all of its own WAN traffic.

Location: Interfaces — WAN — Block private networks — Unchecked

### 2. Disable Reply-To — ENABLED (Critical Fix)

This was the fix that allowed XFlow_Machine (192.168.0.51) to SSH and RDP into lab VMs.

The problem: OPNsense compiles WAN firewall rules with a reply-to directive:

```
reply-to (vtnet0 192.168.0.1)
```

This forces all return traffic back toward the WAN gateway (GE650 at 192.168.0.1).
When XFlow sends traffic into the lab through OPNsense's WAN, reply-to sends responses
back to GE650 instead of directly to XFlow — silently breaking the connection.

An important diagnostic clue: running pfctl -d (disabling all packet filtering)
temporarily made connections work, but they broke again when firewall rules were
re-applied. This was the key indicator that reply-to was the culprit, since the
directive only takes effect when firewall rules are active.

The fix:
```
Firewall — Settings — Advanced — Disable reply-to: Enabled
```

Then flush states and re-enable filtering from the OPNsense shell:
```
pfctl -F states
pfctl -e
```

After this fix, XFlow could reach all lab VMs successfully.

Why this matters: This is a common issue when OPNsense WAN sits on a private subnet
(192.168.0.x) rather than a true public IP. The reply-to behavior is designed for
ISP-facing WAN interfaces, not home lab configurations where traffic crosses between
two private subnets.

## WAN Firewall Rules

Narrow, explicit rules — only what is needed, nothing more. This is the principle
of least privilege applied to firewall policy.

| Rule | Source | Destination | Port | Protocol |
|---|---|---|---|---|
| XFlow to Ubuntu SSH | 192.168.0.51 | 172.16.0.10 | 22 | TCP |
| XFlow to Ubuntu ICMP | 192.168.0.51 | 172.16.0.10 | — | ICMP |
| XFlow to Kali SSH | 192.168.0.51 | 172.16.0.20 | 22 | TCP |
| XFlow to Kali ICMP | 192.168.0.51 | 172.16.0.20 | — | ICMP |
| XFlow to WinServer RDP | 192.168.0.51 | 172.16.0.30 | 3389 | TCP |
| XFlow to WinServer ICMP | 192.168.0.51 | 172.16.0.30 | — | ICMP |

## Emergency Recovery

If a bad firewall rule locks you out of the OPNsense web GUI, recover via
the Proxmox console:

```
pfctl -d        # Disable packet filtering temporarily
pfctl -F states # Flush all connection states
pfctl -e        # Re-enable filtering once rules are fixed
```

## Snapshots

| Snapshot | Description |
|---|---|
| fresh-install-working | Base OPNsense install |
| wizard-complete-working | Setup wizard finished |
| wan-working-full-internet | WAN DHCP working, ping 8.8.8.8 confirmed |
| ssh-routing-replyto-fixed | Reply-to disabled, XFlow access confirmed |

## What I Learned

- OPNsense is FreeBSD-based — uses ifconfig instead of ip addr, pfctl instead of iptables
- Block private networks on WAN must be disabled when WAN is on a home network subnet
- OPNsense reply-to behavior breaks routing in home lab configurations where WAN is
  on a private subnet rather than a true public IP
- pfctl -d as a diagnostic tool — if disabling filtering makes things work, the problem
  is in the firewall rules, not the routing or bridge configuration
- Firewall rule ordering matters — rules are evaluated top to bottom, first match wins
- Principle of least privilege in firewall policy — allow only what is explicitly needed
- How to recover from a firewall lockout using pfctl from the Proxmox console
- Always verify actual VM configuration with qm config before assuming what bridge
  an interface is on
