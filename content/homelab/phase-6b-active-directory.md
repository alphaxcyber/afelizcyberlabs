---

title: Phase 6b - Active Directory Setup
weight: 8
draft: false

---

# Phase 6b - Active Directory Setup 🏛️

With Windows Server 2022 installed and running as ALPHA-DC01, this phase covers promoting it to a Domain Controller, building the full Active Directory structure for alphalab.local, and creating the users, groups, and Group Policy Objects that form the foundation of the lab environment.

---

## Why Active Directory Matters for Cybersecurity

Active Directory is the identity and access management backbone of virtually every enterprise Windows environment. Understanding AD is non-negotiable for both SOC analysts and penetration testers.

From a blue team perspective, AD generates the security logs that SOC analysts live in -- Event ID 4624 (successful logon), 4625 (failed logon), 4768 (Kerberos ticket request), and 4769 (service ticket request) are the signals used to detect credential attacks and lateral movement.

From a red team perspective, AD is the primary attack target. The goal of most penetration tests against Windows environments is domain compromise -- obtaining Domain Admin credentials or a Kerberos ticket that grants full control over every machine in the domain.

Building and administering AD in this lab is the prerequisite for everything that comes after: BloodHound enumeration, Kerberoasting, Pass-the-Hash, and the detection exercises using Wazuh.

---

## Step 1 - Server Rename

Windows Server VMs ship with random generated hostnames. Before promoting to a DC, the server was renamed to follow the lab naming convention.

In enterprise environments, server names follow strict patterns. DC = Domain Controller, FS = File Server, WEB = Web Server, followed by a zero-padded instance number. The ALPHA prefix ties everything back to the AlphaLab brand.

&#x20;   Rename-Computer -NewName "ALPHA-DC01" -Restart



Important: the hostname must be set before AD promotion. After promotion, the hostname is baked into the AD database and changing it requires a complex netdom rename process that can break domain trust relationships.

---

## The O vs Zero Typo - Snapshot Rollback

After the first promotion attempt, verification revealed the hostname had been saved as ALPHA-DCO1 (letter O) rather than ALPHA-DC01 (zero). A single character typo with significant consequences in a production environment.

Since the DC was brand new with nothing built on it, the cleanest fix was:

* Roll back to the pre-promotion Proxmox snapshot named fresh-install-rdp-working
* Rename correctly to ALPHA-DC01 with a zero
* Redo the AD DS installation and promotion from scratch

This was the first real hands-on use of Proxmox snapshot rollback. The rollback completed in seconds and the VM was back to a clean state immediately. It reinforced the value of snapshotting before every significant change, which is now a standing discipline in the lab.

---

## Step 2 - AD DS Role Installation

Active Directory Domain Services is a Windows Server role, not something built into the OS by default. The role was installed with its management tools:

&#x20;   Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools



Result: Success: True, Restart Needed: No

This installs the AD DS binaries and the management tools including Active Directory Users and Computers, Active Directory Sites and Services, and Group Policy Management Console. The server is not yet a DC at this point -- that happens in the next step.

---

## Step 3 - Domain Controller Promotion

With the role installed, the server was promoted to a DC and the alphalab.local forest was created:

&#x20;   Install-ADDSForest `-DomainName "alphalab.local"`
-DomainNetbiosName "ALPHALAB" `-ForestMode "WinThreshold"`
-DomainMode "WinThreshold" `-InstallDns:$true`
-Force:$true



* DSRM password set and stored securely offline
* Server rebooted automatically after promotion
* First boot displayed "Please wait for the Group Policy Client" -- normal behavior on first DC boot
* Login screen changed to ALPHALAB\\Administrator confirming AD was running

Two warnings appeared during promotion, both expected:

* DNS delegation warning: alphalab.local is a private domain with no parent zone, so no delegation is possible. Normal for lab environments.
* NT 4.0 cryptography warning: Server 2022 blocks weak encryption algorithms by default. This is a security feature, not a problem.

---

## Step 4 - Domain Verification

Three verification commands confirmed the domain was healthy:

&#x20;   Get-ADDomain
Get-ADForest
Get-ADDomainController



Key values confirmed:

* DNSRoot: alphalab.local
* NetBIOSName: ALPHALAB
* HostName: ALPHA-DC01.alphalab.local (zero confirmed, not letter O)
* IPv4Address: 172.16.0.30
* IsGlobalCatalog: True
* OperationMasterRoles: SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster

All five FSMO roles on a single DC is correct for a single-DC lab environment. In production, these roles would typically be distributed across multiple DCs for redundancy. The PDCEmulator is the highest-value FSMO role from an attacker's perspective -- it handles password changes and account lockouts domain-wide.

---

## Step 5 - OU Structure

With AD running, the Organizational Unit hierarchy was built. OUs are containers that organize objects in AD and, more importantly, determine which Group Policy Objects apply to which machines and users.

&#x20;   New-ADOrganizationalUnit -Name "AlphaLab" -Path "DC=alphalab,DC=local"
New-ADOrganizationalUnit -Name "Users" -Path "OU=AlphaLab,DC=alphalab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "OU=AlphaLab,DC=alphalab,DC=local"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=AlphaLab,DC=alphalab,DC=local"
New-ADOrganizationalUnit -Name "Admins" -Path "OU=Users,OU=AlphaLab,DC=alphalab,DC=local"
New-ADOrganizationalUnit -Name "Standard" -Path "OU=Users,OU=AlphaLab,DC=alphalab,DC=local"
New-ADOrganizationalUnit -Name "Workstations" -Path "OU=Computers,OU=AlphaLab,DC=alphalab,DC=local"
New-ADOrganizationalUnit -Name "Servers" -Path "OU=Computers,OU=AlphaLab,DC=alphalab,DC=local"



Final structure:

&#x20;   alphalab.local
└── AlphaLab
├── Users
│     ├── Admins
│     └── Standard
├── Computers
│     ├── Workstations
│     └── Servers
└── Groups



The separation between Admins and Standard users is the Principle of Least Privilege in practice. Admin accounts get elevated permissions. Standard users get only what they need for day-to-day work. Service accounts go in Standard because they should never have admin rights unless absolutely required.

---

## Step 6 - Users Created

Four domain accounts were created:

|Name|Username|Type|OU|
|-|-|-|-|
|Alex Feliz|afeliz|Admin|Admins|
|John Smith|jsmith|Standard User|Standard|
|Maria Garcia|mgarcia|Standard User|Standard|
|SVC Backup|svc\_backup|Service Account|Standard|

afeliz was added to Domain Admins:

&#x20;   Add-ADGroupMember -Identity "Domain Admins" -Members "afeliz"



The service account svc\_backup represents a common real-world pattern -- automated backup software runs as a service account with read access to file shares. Service accounts are a primary attack target because they often have elevated privileges, rarely change their passwords, and are easy to Kerberoast.

---

## Step 7 - Security Groups

Three security groups were created in the Groups OU:

&#x20;   New-ADGroup -Name "IT Admins" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=AlphaLab,DC=alphalab,DC=local"
New-ADGroup -Name "SOC Analysts" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=AlphaLab,DC=alphalab,DC=local"
New-ADGroup -Name "Standard Users" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=AlphaLab,DC=alphalab,DC=local"



The correct approach is Role Based Access Control: assign permissions to groups, then add users to groups. Never assign permissions directly to individual user accounts. This way, when someone joins or leaves a team, you change their group membership rather than hunting down every permission they were individually granted.

---

## Step 8 - Group Policy Objects

Three GPOs were created and linked to the domain root:

**AlphaLab Password Policy**

* 12-character minimum password length
* Enforced: True

**AlphaLab Account Lockout Policy**

* Account lockout after 5 failed attempts
* Enforced: True
* Note: setting this too aggressively (1 or 2 attempts) enables a denial-of-service attack where an attacker intentionally locks out accounts across the domain

**AlphaLab Security Audit Policy**

* Audit value 3 = log both success and failure events for logon events and account management
* Enforced: True
* This is the foundation for Wazuh SIEM integration -- without audit policy enabled, attacks happen in complete silence

Final GPO processing order at domain root:

&#x20;   Order 1: Default Domain Policy       (built-in)
Order 2: AlphaLab Password Policy    (Enforced)
Order 3: AlphaLab Account Lockout    (Enforced)
Order 4: AlphaLab Security Audit     (Enforced)



Key Windows Event IDs generated by the audit policy that SOC analysts watch:

|Event ID|Meaning|
|-|-|
|4624|Successful logon|
|4625|Failed logon|
|4648|Logon using explicit credentials|
|4720|User account created|
|4732|User added to security group|
|4768|Kerberos ticket request (TGT)|
|4769|Kerberos service ticket request|
|4771|Kerberos pre-authentication failed|

---

## Snapshots Taken

|Name|Description|
|-|-|
|dc-promoted-clean|ALPHA-DC01 promoted, AD DS and DNS green, pre-Windows Update|
|dc-post-update-clean|Post-update, all AD services healthy, ready for OU and user build|

---

## Current Lab State

|VM|Hostname|IP|Role|
|-|-|-|-|
|VM 100|OPNsense|172.16.0.1|Firewall and gateway|
|VM 101|Ubuntu Server|172.16.0.10|Linux server|
|VM 102|Kali Linux|172.16.0.20|Attack workstation|
|VM 103|ALPHA-DC01|172.16.0.30|Domain Controller|

---

## What Is Next

With the domain built and all policies in place, the next step is joining the Linux VMs to alphalab.local. Both Ubuntu Server and Kali Linux will be domain members, which is what enables domain-authenticated attacks using tools like BloodHound and Impacket in later phases.


## What I Learned

- Active Directory is the identity backbone of enterprise Windows environments and the primary target in most penetration tests
- The difference between installing an AD DS role and promoting a server to a Domain Controller -- two separate steps
- Why the hostname must be set before AD promotion -- changing it afterward is a complex and risky process
- FSMO roles and why the PDCEmulator is a high-value attack target
- The Principle of Least Privilege -- admin accounts, standard accounts, and service accounts each get only the access they need
- Role Based Access Control -- assign permissions to groups, never to individual users
- Why service accounts are a primary Kerberoasting target -- elevated privileges, infrequent password changes
- Group Policy as both a defensive hardening tool and a high-value target for attackers
- Account lockout policy can be weaponized as a denial-of-service attack if set too aggressively
- Security audit policy is the foundation of SIEM detection -- without it attacks happen in silence
- The key Windows Event IDs that SOC analysts monitor: 4624, 4625, 4648, 4720, 4732, 4768, 4769, 4771
- Snapshot rollback as a recovery tool -- the O vs zero typo was resolved in seconds by rolling back to a clean snapshot
