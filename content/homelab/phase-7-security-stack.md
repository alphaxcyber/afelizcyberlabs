---
title: Phase 7 - Security Stack
weight: 9
---

# Phase 7 — Security Stack 🛡️

## Overview

With the core infrastructure in place — OPNsense firewall, Ubuntu Server, Kali Linux,
and Windows Server — the next phase of the lab focuses on building out the security
monitoring and operations stack. This is where the lab transitions from a basic
virtualization environment into a functional Security Operations Center (SOC) lab.

This phase is currently in progress. Pages will be added as each component is
deployed and configured.

## Planned VM Deployments

| VM ID | System | IP | RAM | Disk | Purpose |
|---|---|---|---|---|---|
| 104 | Security Onion | 172.16.0.40 | 16GB | 100GB | NSM / IDS / Elastic |
| 105 | Wazuh SIEM | 172.16.0.50 | 4GB | 50GB | Log analysis / alerts |
| 106 | Metasploitable 3 | 172.16.0.60 | 2GB | 40GB | Vulnerable target |

## Security Onion

Security Onion is a free and open-source Linux distribution for threat hunting,
network security monitoring (NSM), and log management. It bundles several
powerful tools into a single deployable platform:

- **Zeek** (formerly Bro) — network traffic analyzer that generates detailed
  connection logs, DNS queries, HTTP requests, SSL certificates, and more
- **Suricata** — high-performance intrusion detection and prevention system (IDS/IPS)
  with signature-based and anomaly-based detection
- **Elastic Stack** — Elasticsearch, Logstash, and Kibana for log ingestion,
  storage, and visualization
- **TheHive integration** — incident response case management
- **Playbook** — detection rule management

Security Onion will be deployed with a network tap on vmbr1 to capture all
traffic between lab VMs. This allows every attack generated from Kali to be
captured, analyzed, and used to build detection rules.

### Why 16GB RAM

Security Onion is memory-intensive. Elasticsearch alone requires significant
heap allocation to function properly. 16GB is the recommended minimum for a
functional standalone deployment. Allocating less results in poor performance
and frequent out-of-memory conditions.

## Wazuh SIEM

Wazuh is an open-source security platform that provides:

- **Host-based intrusion detection (HIDS)** — agent installed on each VM monitors
  file integrity, process execution, and system calls
- **Log analysis** — collects and analyzes logs from all lab systems
- **Vulnerability detection** — scans agents for known CVEs
- **Active response** — can automatically block IPs or kill processes in response
  to detected threats
- **Compliance monitoring** — maps findings to frameworks like MITRE ATT&CK,
  PCI DSS, and HIPAA

Wazuh agents will be deployed on Ubuntu Server, Kali, and Windows Server 2022.
This means every command run on those systems, every login attempt, and every
file change will be forwarded to the Wazuh manager for analysis and alerting.

## Metasploitable 3

Metasploitable 3 is an intentionally vulnerable virtual machine built by Rapid7
(the makers of Metasploit) for practicing exploitation in a safe, legal environment.
It contains numerous deliberately configured vulnerabilities across services like
SMB, HTTP, FTP, and others.

Using Metasploitable as a target allows for:

- Practicing Metasploit Framework exploitation techniques
- Testing detection rules — does Security Onion or Wazuh alert when Metasploit
  runs a specific exploit?
- Building attack/detect/respond workflows
- OSCP exam preparation — Metasploitable vulnerabilities mirror real exam targets

## Planned Lab Expansion

| VM | Purpose | Phase |
|---|---|---|
| TrueNAS Scale | NAS / private cloud storage | 3 |
| Jellyfin | 4K media streaming | 3 |
| Whonix Gateway | Tor network gateway | 4 |
| Whonix Workstation | Anonymous research workstation | 4 |
| Ubuntu ML Server | AI / security research | 4 |
| TheHive + Cortex | SOC incident response platform | 4 |

## Weekly Lab Curriculum (When Fully Built)

Once the full security stack is deployed, the lab will support a structured
weekly training curriculum:

| Day | Focus |
|---|---|
| Monday | AD attacks + Security Onion detection |
| Tuesday | Web application attacks + detection rules |
| Wednesday | REST DAY |
| Thursday | Cloud security (AWS) |
| Friday | VulnHub / HackMyVM methodology |
| Saturday | Flexible / catch-up |
| Sunday | REST DAY |

## The Attack-Detect-Respond Workflow

The core learning loop this security stack enables:

```
1. ATTACK
   Kali Linux generates malicious traffic
   (port scan, exploit attempt, credential spray)
        |
        v
2. DETECT
   Security Onion captures network traffic
   Wazuh agent captures host activity
   Alerts generated in dashboards
        |
        v
3. ANALYZE
   Investigate alerts in Kibana / Wazuh UI
   Trace attack path through logs
   Identify indicators of compromise (IOCs)
        |
        v
4. RESPOND
   Document findings in TheHive case
   Write or tune detection rules
   Harden the vulnerable system
        |
        v
5. REPEAT
   Run the same attack -- does the new rule catch it?
```

This is the same workflow used in real SOC environments. Practicing it in a
controlled lab builds the muscle memory and analytical instincts that translate
directly to Tier 1 and Tier 2 SOC analyst roles.

## Current Status

| Component | Status |
|---|---|
| Security Onion | Planned — deployment pending |
| Wazuh SIEM | Planned — deployment pending |
| Metasploitable 3 | Planned — deployment pending |
| TrueNAS Scale | Planned — awaiting storage pricing |
| Ubuntu ML Server | Planned — future phase |

This page will be updated with full deployment documentation as each component
is built out. Check back for new additions as the lab evolves.
