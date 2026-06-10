---
title: Phase 2 - Network Configuration
weight: 3
---

# Phase 2 — Network Configuration 🌐

## Physical Network Layout

| Device | Role | IP |
|---|---|---|
| Xfinity Gateway | Upstream internet | — |
| TP-Link GE650 | Home router | 192.168.0.1 |
| Netgear GS308 | Dumb switch | — |
| HP Z440 (Proxmox) | Lab server | 192.168.0.10 |
| XFlow_Machine | Main workstation | 192.168.0.51 |
| OPNsense WAN | Firewall WAN | 192.168.0.207 |

## WiFi Networks

| SSID | Band | Purpose |
|---|---|---|
| standard_issue | 2.4/5GHz | Main home network |
| apex_issue | 6GHz | High speed devices |
| cold_zone | 2.4/5GHz | IoT devices (isolated) |

## Proxmox Bridge Architecture

Proxmox uses Linux bridges — think of them as virtual switches inside the hypervisor.
VMs plug into bridges instead of physical NICs directly.

| Bridge | Physical NIC | Purpose |
|---|---|---|
| vmbr0 | eno1 | Proxmox management + OPNsense WAN (192.168.0.x) |
| vmbr1 | none | Isolated lab network (172.16.0.x) |
| vmbr2 | enp12s0 | Available for future use |

```
/etc/network/interfaces:

auto vmbr0
iface vmbr0 inet static
        address 192.168.0.10/24
        gateway 192.168.0.1
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0

auto vmbr1
iface vmbr1 inet manual
        bridge-ports none        # No physical port — isolated lab network
        bridge-stp off
        bridge-fd 0

auto vmbr2
iface vmbr2 inet manual
        bridge-ports enp12s0     # Available for future use
        bridge-stp off
        bridge-fd 0
```

## Problem 1 — OPNsense WAN ARP Conflict

Initially OPNsense WAN (net0) was assigned to vmbr0 alongside Proxmox management,
both sharing eno1. This caused ARP conflicts — the GE650 saw two different MAC
addresses on the same wire and refused to respond to OPNsense DHCP requests.

Diagnosis:
```
tcpdump -i vtnet1 port 67 or port 68   # DHCP packets leaving but no response
arp -a                                  # Incomplete ARP entries confirmed
```

Fix: Installed the Fenvi Dual Port Intel I226-V NIC. Created vmbr2 on enp12s0 and
temporarily moved OPNsense WAN to it — dedicated physical wire, no ARP conflicts.
OPNsense WAN received 192.168.0.207 via DHCP and internet access was confirmed:
ping 8.8.8.8 → 0% packet loss.

Additionally, OPNsense had its internal interfaces swapped (vtnet0/vtnet1 assigned
backwards). Verified by MAC address using ifconfig vtnet1 in the OPNsense shell
and corrected via console Option 1.

## Problem 2 — SSH and RDP Access Failing from XFlow

After the VMs were built and OPNsense had internet access, attempts to SSH and RDP
from XFlow_Machine (192.168.0.51) into lab VMs (172.16.0.x) were failing silently.
Pings and connections would not complete.

The initial suspicion was that having OPNsense WAN on vmbr2 (enp12s0) while Proxmox
management was on vmbr0 (eno1) was creating two separate Layer 2 broadcast domains
within the same 192.168.0.0/24 subnet — causing the GE650 to be unable to route
return traffic correctly.

The proposed fix was to move OPNsense WAN back to vmbr0 so both Proxmox and OPNsense
WAN would be on the same bridge, behaving like two devices on the same virtual switch.

## Important Discovery — OPNsense WAN Was Already on vmbr0

Before making any changes, the actual OPNsense VM configuration was verified from
the Proxmox shell:

```
qm config 100 | grep net
```

Result:
```
net0: virtio=BC:24:11:8A:88:5F,bridge=vmbr0
net1: virtio=BC:24:11:90:CA:B7,bridge=vmbr1
```

This confirmed that OPNsense WAN (net0) was already on vmbr0 — not vmbr2.
The assumption that vmbr2 was the active WAN bridge was incorrect.
vmbr2 had been used temporarily during the ARP conflict troubleshooting,
but the configuration had already been reverted.

This meant vmbr2 was never the actual problem. The real issue was something else entirely.

## The Real Problem — OPNsense Reply-To Behavior

OPNsense compiles WAN firewall rules with a reply-to directive that forces return
traffic back toward the WAN gateway. In this case:

```
reply-to (vtnet0 192.168.0.1)
```

When XFlow (192.168.0.51) sent traffic into the lab through OPNsense's WAN interface,
the reply-to directive forced all return traffic back toward the GE650 (192.168.0.1)
instead of routing it directly back to XFlow. This silently broke every connection
attempt.

This explained a previously confusing symptom: running pfctl -d (which disables
all packet filtering) temporarily made connections work, but they broke again the
moment firewall rules were re-applied. The reply-to directive only takes effect
when firewall rules are active.

Fix: OPNsense Web GUI — Firewall — Settings — Advanced — Disable reply-to: Enabled

Then connection states were flushed and filtering was re-enabled from the OPNsense shell:
```
pfctl -F states
pfctl -e
```

After this, XFlow could successfully reach all lab VMs:
```
ping 172.16.0.10       # Ubuntu — success
ssh alphax@172.16.0.10 # Ubuntu SSH — success
```

## Final Working Architecture

```
Internet
    |
Xfinity Gateway
    |
TP-Link GE650 (192.168.0.1)
    |
    |-- eno1 -- vmbr0 -- Proxmox management (192.168.0.10)
    |                 -- OPNsense WAN (192.168.0.207)
    |
    |           vmbr1 -- OPNsense LAN (172.16.0.1)
    |                         |
    |           +--------------+--------------+
    |           |              |              |
    |     Ubuntu Server    Kali Linux    Win Server
    |     172.16.0.10     172.16.0.20   172.16.0.30
    |
    |-- enp12s0 -- vmbr2 -- (available for future use)
```

## XFlow Persistent Route

For XFlow_Machine (192.168.0.51) to reach lab VMs, a persistent static route
was added pointing toward OPNsense WAN:

```
route -p add 172.16.0.0 mask 255.255.255.0 192.168.0.207
```

This tells XFlow: any traffic destined for 172.16.0.x should go to OPNsense
at 192.168.0.207, not the default gateway.

## GE650 DHCP Reservations

Static IP reservations were added to the GE650 to ensure consistent addressing:

| IP | Device | MAC |
|---|---|---|
| 192.168.0.10 | Proxmox | a0:8c:fd:d8:98:74 |
| 192.168.0.51 | XFlow_Machine | reserved |
| 192.168.0.207 | OPNsense WAN | bc:24:11:8a:88:5f |

## What I Learned

- Always verify actual configuration before making changes — qm config 100 | grep net
  revealed the real state and prevented an unnecessary bridge migration
- ARP conflicts occur when two devices share the same physical NIC on the same bridge
- OPNsense reply-to behavior forces return traffic toward the WAN gateway — this breaks
  routing in home lab configurations where WAN is on a private subnet
- pfctl -d temporarily disabling filtering was the clue that reply-to was the culprit,
  since reply-to only applies when firewall rules are active
- Same IP subnet does not guarantee same Layer 2 broadcast domain when traffic is
  separated across different bridges and NIC paths
- vmbr2 and enp12s0 remain available for future lab expansion such as a DMZ segment,
  Security Onion mirror port, or alternate WAN design
