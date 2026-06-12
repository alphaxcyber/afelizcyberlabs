---
title: "Debug Mission Log #1 - OPNsense reply-to Breaking SSH Return Path"
weight: 1
draft: false
date: 2026-06-09
---

# Debug Mission Log #1 🛠️
## OPNsense reply-to Breaking SSH Return Path

**Environment:** Proxmox homelab · OPNsense 25.1 · HP Z440  
**Symptom:** SSH and RDP from home network into lab VMs failing silently  
**Root Cause:** OPNsense `reply-to` directive misrouting return traffic  
**Status:** ✅ Resolved

---

## The Setup

At this point in the lab build, the network looked like this:

- **XFlow_Machine** (192.168.0.51) — main workstation on home network
- **OPNsense WAN** (192.168.0.207) — firewall sitting on the same home subnet
- **Lab VMs** (172.16.0.x) — isolated behind OPNsense LAN
- **Static route on XFlow:** `172.16.0.0/24 → 192.168.0.207`

OPNsense had internet access, VMs were reachable from inside the lab, and everything looked correct on paper.

---

## The Symptom

Attempting SSH from XFlow into Ubuntu Server:

```bash
ssh alphax@172.16.0.10
```

Connection would hang and time out. No error — just silence. RDP into Windows Server (172.16.0.30) behaved the same way.

Packets were clearly leaving XFlow and the static route was correct. Something was happening on the return path.

---

## Diagnosing the Return Path

The key diagnostic clue came from OPNsense itself. Running this from the OPNsense shell temporarily disabled packet filtering:

```bash
pfctl -d
```

After disabling filtering, SSH worked immediately. Re-enabling filtering broke it again:

```bash
pfctl -e
```

That told the story clearly — the problem wasn't routing, it wasn't the VMs, it wasn't the static route. The problem was inside OPNsense's firewall rule compilation.

---

## Root Cause: reply-to

OPNsense automatically compiles WAN firewall rules with a `reply-to` directive:

```
reply-to (vtnet0 192.168.0.1)
```

This directive tells OPNsense to force all return traffic back toward the WAN gateway — in this case, the TP-Link GE650 at 192.168.0.1.

**Why this breaks the lab setup:**

In a standard ISP-facing deployment, `reply-to` makes sense. If traffic arrives on the WAN from the internet, replies should go back out the WAN. But in this lab, OPNsense WAN sits on a private subnet (192.168.0.x) — the same subnet as XFlow_Machine. When XFlow sends SSH traffic to 172.16.0.10, the packets enter through OPNsense WAN. The `reply-to` rule then forces return traffic toward the GE650 gateway instead of back to XFlow directly, breaking the return path.

The traffic flow looked like this:

```
XFlow (192.168.0.51)
  → OPNsense WAN (192.168.0.207)
    → Ubuntu (172.16.0.10)
      → return traffic forced to GE650 (192.168.0.1)  ❌
```

Instead of:

```
XFlow (192.168.0.51)
  → OPNsense WAN (192.168.0.207)
    → Ubuntu (172.16.0.10)
      → return traffic back to XFlow (192.168.0.51)   ✅
```

---

## The Fix

**OPNsense Web GUI:**

```
Firewall → Settings → Advanced → Disable reply-to → ✅ Enable
```

Save and apply. Then flush existing connection states and re-enable filtering from the OPNsense shell:

```bash
pfctl -F states
pfctl -e
```

`pfctl -F states` clears any stale connections that were built with the old reply-to behavior. Without this step, existing sessions may continue to misbehave even after the setting change.

---

## Result

```bash
ssh alphax@172.16.0.10
# Connected ✅

ping 172.16.0.10
# 0% packet loss ✅
```

RDP into Windows Server (172.16.0.30) also working.

---

## What I Learned

**reply-to is designed for ISP-facing WAN interfaces.** When OPNsense WAN lives on a private subnet and you're routing between two RFC 1918 networks, reply-to actively breaks the return path by introducing an unnecessary hop through the upstream gateway.

**pfctl -d is a powerful diagnostic tool.** Temporarily disabling the firewall to isolate whether a connectivity issue is routing-related or firewall-related is a fast way to narrow down the problem. If things work with filtering disabled, the firewall rules are your problem.

**State tables persist after rule changes.** Always flush states with `pfctl -F states` after significant firewall configuration changes. Old sessions carry the old rule behavior.

---

## Environment Reference

| Component | Detail |
|---|---|
| Hypervisor | Proxmox VE 8.4 |
| Firewall | OPNsense 25.1 |
| OPNsense WAN | vtnet0 → vmbr0 → 192.168.0.207 |
| OPNsense LAN | vtnet1 → vmbr1 → 172.16.0.1 |
| XFlow_Machine | 192.168.0.51 |
| Ubuntu Server | 172.16.0.10 |
| Static Route | 172.16.0.0/24 via 192.168.0.207 |
