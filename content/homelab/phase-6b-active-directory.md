---
title: Phase 6b - Active Directory Setup
weight: 8
---

# Phase 6b — Active Directory Setup 🛡️

**Date:** June 10, 2026
**Lab Environment:** HP Z440 · Proxmox VE 8.4 · OPNsense · Windows Server 2022

---

## Overview

With OPNsense running as my lab firewall, Ubuntu Server and Kali Linux already on the network, the next major milestone for AlphaLab was promoting Windows Server 2022 to a Domain Controller and standing up a full Active Directory environment. This post covers everything from renaming the server to building out Organizational Units, users, groups, and Group Policy Objects — and the one naming mistake I caught just in time.

By the end of this session, AlphaLab had a fully functional Active Directory domain: alphalab.local.

---

## Why Active Directory Matters for Cybersecurity

Before getting into the steps, it is worth understanding why building AD from scratch is one of the most valuable things you can do in a cybersecurity homelab.

Almost every enterprise environment in the world runs Active Directory. As a SOC Analyst you will be investigating AD logs constantly — failed logins, privilege escalation attempts, and lateral movement between machines. As a penetration tester, AD is almost always the primary target. Tools like BloodHound, Mimikatz, and Impacket exist almost entirely to attack AD environments.

You cannot work in enterprise cybersecurity without understanding AD deeply. Building it yourself from the inside out is far more valuable than just reading about it.

---

## Starting Point

Going into this session, VM 103 was running Windows Server 2022 Standard Evaluation at 172.16.0.30 on the lab network. RDP was working from my main workstation (XFlow) via OPNsense, VirtIO drivers were installed, and a clean snapshot (fresh-install-rdp-working) was in place.

One thing still needed: the server had a default random hostname assigned during installation. Before touching Active Directory, that had to be fixed.

---

## Step 1 — Renaming the Server

Hostname naming conventions matter in enterprise environments. The standard pattern is a role abbreviation plus a number:

- DC01, DC02 — Domain Controllers
- FS01 — File Servers
- WEB01 — Web Servers

For AlphaLab, I went with ALPHA-DC01 to keep it consistent with the lab naming theme.

    Rename-Computer -NewName "ALPHA-DC01" -Restart

After the reboot, verify it took:

    hostname

**Important:** This has to be done before AD promotion. Once a server is a Domain Controller, the hostname becomes baked into the AD database and renaming it requires a multi-step netdom process. Renaming before promotion keeps things clean.

---

## The Mistake I Caught Just in Time

After the first promotion attempt, running Get-ADDomainController showed ALPHA-DCO1 in the output — with the letter O instead of a zero. A subtle but permanent mistake inside AD.

Since the DC was brand new with nothing built on it yet, the cleanest fix was to roll back to the pre-promotion snapshot in Proxmox, rename the computer correctly, and redo the promotion. This took about 10 minutes and avoided any risk of a botched DC rename leaving AD in an inconsistent state.

**Lesson:** Always double-check O vs 0 when naming servers. And always take snapshots before major changes — they are what made this a 10-minute fix instead of a major problem.

---

## Step 2 — Installing the AD DS Role

With the correct hostname confirmed, install the Active Directory Domain Services role:

    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

The -IncludeManagementTools flag installs the GUI tools and PowerShell modules alongside the role itself. When it finishes you should see Success: True and Restart Needed: No.

---

## Step 3 — Promoting to Domain Controller

This is the command that transforms a regular Windows Server into a Domain Controller and creates a brand new Active Directory forest:

    Install-ADDSForest `
      -DomainName "alphalab.local" `
      -DomainNetbiosName "ALPHALAB" `
      -ForestMode "WinThreshold" `
      -DomainMode "WinThreshold" `
      -InstallDns:$true `
      -Force:$true

What each parameter does:

- -DomainName "alphalab.local" — The fully qualified domain name. .local means private, internal only
- -DomainNetbiosName "ALPHALAB" — The short legacy name used for ALPHALAB\username style logins
- -ForestMode "WinThreshold" — Windows Server 2016 functional level, highest compatible with 2022
- -InstallDns:$true — Installs DNS server automatically, required since AD depends entirely on DNS
- -Force:$true — Suppresses confirmation prompts

You will be prompted for a DSRM password (Directory Services Restore Mode). This is an emergency recovery password used only if AD itself becomes corrupted. It is separate from your Administrator password. Write it down and store it somewhere safe.

The server will reboot automatically when promotion completes.

### What Those Warnings Mean

During promotion you will see two warnings that look alarming but are completely normal:

**DNS delegation warning:** alphalab.local is a private domain that does not exist on the public internet, so there is no parent DNS zone to delegate from. In a private homelab this is expected — ignore it.

**NT 4.0 cryptography warning:** Windows Server 2022 blocks old weak encryption algorithms by default. This is actually a good thing — your DC is more secure out of the box.

### First Boot After Promotion

The first login after promotion will show a "Please wait for the Group Policy Client" screen. Do not touch it. The server is doing significant first-time domain setup work behind the scenes — building the AD database, initializing SYSVOL, registering DNS records. On a VM this takes 2-5 minutes and resolves on its own.

When the login screen appears, you will notice it now shows ALPHALAB\Administrator instead of just Administrator. That is your first visual confirmation that Active Directory is running.

---

## Step 4 — Verifying the Domain

Three commands confirm everything came up healthy:

    Get-ADDomain
    Get-ADForest
    Get-ADDomainController

Key things to verify in the output:

- DNSRoot: alphalab.local
- NetBIOSName: ALPHALAB
- HostName: ALPHA-DC01.alphalab.local (zero, not letter O)
- IPv4Address: 172.16.0.30
- IsGlobalCatalog: True
- OperationMasterRoles: SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster

### FSMO Roles

FSMO stands for Flexible Single Master Operations. These are five special roles that only one DC can hold at a time, handling operations that cannot be performed simultaneously on multiple servers:

- SchemaMaster — Controls changes to the AD schema
- DomainNamingMaster — Controls adding and removing domains from the forest
- PDCEmulator — Handles password changes, time sync, and Group Policy
- RIDMaster — Issues blocks of security IDs for new objects
- InfrastructureMaster — Tracks object references across domains

In a single-DC lab, ALPHA-DC01 holds all five — which is correct. In large enterprises these are sometimes spread across multiple DCs for redundancy. The PDCEmulator is especially valuable to attackers; compromising it gives enormous power over the domain.

---

## Step 5 — Running Windows Update

Before building anything in AD, take a snapshot and run Windows Update. The order matters:

    Verify working state → Snapshot → Apply updates → Verify again → Snapshot

This is a habit worth building. Updates can occasionally break things, and a snapshot means any problem is a 30-second rollback rather than a rebuild.

After the update reboot, verify the four critical AD services are still running:

    Get-Service adws, kdc, netlogon, dns

- adws — Active Directory Web Services, enables PowerShell AD commands
- kdc — Kerberos Distribution Center, the authentication engine of AD
- netlogon — Handles domain join requests and DC location
- dns — DNS Server, AD is completely dependent on this

---

## Step 6 — Building the OU Structure

With the domain healthy and updated, it is time to build the skeleton of the directory. Organizational Units (OUs) are the folder structure that organizes everything in AD.

**Key concept:** OUs are for organization. Groups are for permissions. Location in an OU does not grant access — group membership does.

The structure built for AlphaLab mirrors a real small enterprise:

    alphalab.local
    └── AlphaLab
          ├── Users
          │     ├── Admins
          │     └── Standard
          ├── Computers
          │     ├── Workstations
          │     └── Servers
          └── Groups

    New-ADOrganizationalUnit -Name "AlphaLab" -Path "DC=alphalab,DC=local"
    New-ADOrganizationalUnit -Name "Users" -Path "OU=AlphaLab,DC=alphalab,DC=local"
    New-ADOrganizationalUnit -Name "Computers" -Path "OU=AlphaLab,DC=alphalab,DC=local"
    New-ADOrganizationalUnit -Name "Groups" -Path "OU=AlphaLab,DC=alphalab,DC=local"
    New-ADOrganizationalUnit -Name "Admins" -Path "OU=Users,OU=AlphaLab,DC=alphalab,DC=local"
    New-ADOrganizationalUnit -Name "Standard" -Path "OU=Users,OU=AlphaLab,DC=alphalab,DC=local"
    New-ADOrganizationalUnit -Name "Workstations" -Path "OU=Computers,OU=AlphaLab,DC=alphalab,DC=local"
    New-ADOrganizationalUnit -Name "Servers" -Path "OU=Computers,OU=AlphaLab,DC=alphalab,DC=local"

---

## Step 7 — Creating Users

A realistic lab environment needs different account types:

    New-ADUser -Name "Alex Feliz" -SamAccountName "afeliz" -UserPrincipalName "afeliz@alphalab.local" -Path "OU=Admins,OU=Users,OU=AlphaLab,DC=alphalab,DC=local" -AccountPassword (ConvertTo-SecureString "Lab@dmin123!" -AsPlainText -Force) -Enabled $true

    New-ADUser -Name "John Smith" -SamAccountName "jsmith" -UserPrincipalName "jsmith@alphalab.local" -Path "OU=Standard,OU=Users,OU=AlphaLab,DC=alphalab,DC=local" -AccountPassword (ConvertTo-SecureString "User@pass123!" -AsPlainText -Force) -Enabled $true

    New-ADUser -Name "Maria Garcia" -SamAccountName "mgarcia" -UserPrincipalName "mgarcia@alphalab.local" -Path "OU=Standard,OU=Users,OU=AlphaLab,DC=alphalab,DC=local" -AccountPassword (ConvertTo-SecureString "User@pass123!" -AsPlainText -Force) -Enabled $true

    New-ADUser -Name "SVC Backup" -SamAccountName "svc_backup" -UserPrincipalName "svc_backup@alphalab.local" -Path "OU=Standard,OU=Users,OU=AlphaLab,DC=alphalab,DC=local" -AccountPassword (ConvertTo-SecureString "Backup@svc123!" -AsPlainText -Force) -Enabled $true

**Why a service account?** Service accounts are used by applications and automated processes rather than humans. In real environments they frequently have weak passwords that never expire — making them a prime target for privilege escalation. The common attack chain:

    Compromise standard user
        ↓
    Find misconfigured service account
        ↓
    Escalate privileges
        ↓
    Reach Domain Admin
        ↓
    Own the domain

---

## Step 8 — Adding afeliz to Domain Admins

Having a user in the Admins OU does not grant admin rights. What grants elevated privileges is group membership:

    Add-ADGroupMember -Identity "Domain Admins" -Members "afeliz"

Domain Admins members have full administrative control over every machine in the domain. In real environments, membership is strictly controlled, heavily audited, and a primary target for attackers.

---

## Step 9 — Creating Security Groups

    New-ADGroup -Name "IT Admins" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=AlphaLab,DC=alphalab,DC=local" -Description "IT Administrative Staff"

    New-ADGroup -Name "SOC Analysts" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=AlphaLab,DC=alphalab,DC=local" -Description "Security Operations Center Analysts"

    New-ADGroup -Name "Standard Users" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=AlphaLab,DC=alphalab,DC=local" -Description "Standard domain users"

    Add-ADGroupMember -Identity "IT Admins" -Members "afeliz"
    Add-ADGroupMember -Identity "SOC Analysts" -Members "afeliz"
    Add-ADGroupMember -Identity "Standard Users" -Members "jsmith", "mgarcia", "svc_backup"

afeliz gets both IT Admins and SOC Analysts. Standard users get Standard Users only — Principle of Least Privilege in action.

---

## Step 10 — Group Policy Objects

Group Policy pushes security settings down to users and computers automatically. Three GPOs were created for AlphaLab:

### Password Policy

    New-GPO -Name "AlphaLab Password Policy" -Comment "Domain-wide password security policy"
    New-GPLink -Name "AlphaLab Password Policy" -Target "DC=alphalab,DC=local" -Enforced Yes

Enforces a 12-character minimum. NIST guidelines recommend 12-16 characters as the enterprise minimum — length matters more than complexity for resisting brute force.

### Account Lockout Policy

    New-GPO -Name "AlphaLab Account Lockout Policy" -Comment "Brute force protection"
    New-GPLink -Name "AlphaLab Account Lockout Policy" -Target "DC=alphalab,DC=local" -Enforced Yes

Locks accounts after 5 failed attempts — stopping automated brute force tools cold.

### Security Audit Policy

    New-GPO -Name "AlphaLab Security Audit Policy" -Comment "Enable security event logging for SOC monitoring"
    New-GPLink -Name "AlphaLab Security Audit Policy" -Target "DC=alphalab,DC=local" -Enforced Yes

Without audit policy, attacks happen in silence. With it enabled, every login attempt and account change gets written to the Security Event Log — the foundation of SOC monitoring.

Windows Event IDs every SOC analyst should know:

- 4624 — Successful logon
- 4625 — Failed logon, brute force shows up here
- 4648 — Logon with explicit credentials, Pass-the-Hash indicator
- 4720 — User account created
- 4732 — User added to privileged group
- 4768 — Kerberos TGT requested
- 4769 — Kerberos service ticket requested
- 4771 — Kerberos pre-authentication failed

### Final GPO State

    Order 1: Default Domain Policy      (built-in, Enforced: False)
    Order 2: AlphaLab Password Policy   (Enforced: True)
    Order 3: AlphaLab Account Lockout   (Enforced: True)
    Order 4: AlphaLab Security Audit    (Enforced: True)

---

## How It All Connects

    User logs in
        ↓
    Kerberos (KDC) validates credentials
        ↓
    Group membership checked — determines what they can access
        ↓
    GPOs applied — enforce security settings on their session
        ↓
    Audit policy logs the event — written to Security Event Log
        ↓
    Future: Wazuh SIEM collects that log — generates alerts

Every component of that chain is now in place in AlphaLab.

---

## Snapshots Taken This Session

- dc-promoted-clean — ALPHA-DC01 promoted, AD DS and DNS green, pre-update
- dc-post-update-clean — Post Windows Update, all services healthy, ready for OU and user build

---

## What's Next

- Join Kali Linux and Ubuntu Server to the alphalab.local domain
- Configure fine-grained password policies for service accounts
- Deploy Wazuh SIEM to start collecting audit logs
- Begin Active Directory attack and defense exercises — BloodHound enumeration, privilege escalation paths, and detection

The lab is starting to look like a real enterprise environment.

---

AlphaLab is a cybersecurity homelab build series documenting the journey toward a career in cybersecurity. Follow along at afelizcyberlabs.com.
