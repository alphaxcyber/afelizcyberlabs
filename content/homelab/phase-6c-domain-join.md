---
title: Phase 6c - Joining Linux VMs to the Domain
weight: 9
draft: false
---

# Phase 6c - Joining Linux VMs to the Domain 🖧

With Active Directory fully built on ALPHA-DC01, the next step was bringing the Linux VMs into the domain. This phase covers joining both Ubuntu Server and Kali Linux to alphalab.local, cleaning up computer objects in AD, and the workaround required to get Kali joined despite its missing sssd packages.

---

## Why Join Linux Machines to Active Directory?

In enterprise environments, Linux servers are commonly domain-joined so they can authenticate users via Kerberos, apply Group Policy equivalents, and be managed centrally. For this lab, joining both VMs serves a different but equally important purpose -- it lets us practice realistic attack and defense scenarios.

- Ubuntu Server (ALPHA-SRV-01) represents an internal domain-joined server, a common target for lateral movement after initial access
- Kali Linux (KALI-LINUX) represents a domain-joined attacker workstation, used for authenticated attacks against AD such as BloodHound enumeration, Kerberoasting, and Pass-the-Hash

Having both machines inside the domain trust boundary is what makes tools like BloodHound and Impacket work at full capability.

---

## Step 1 - Fix the Ubuntu Hostname

Ubuntu had previously joined the domain as UBUNTU-SERVER-0, which was a truncated version of the intended name UBUNTU-SERVER-01. This happened because NetBIOS names are limited to 15 characters. The correct fix was to rename the host before rejoining.

On Ubuntu Server, the hostname was updated using hostnamectl:

    sudo hostnamectl set-hostname ALPHA-SRV-01

Verified with:

    hostnamectl

Output confirmed:

    Static hostname: ALPHA-SRV-01

Note: the shell prompt still showed the old hostname for the remainder of that session. This is expected -- the prompt reflects the hostname that was active when the session started, and will update automatically on next login.

---

## Step 2 - Rejoin Ubuntu with the Correct Hostname

With the hostname corrected, the old domain membership was dropped and the VM rejoined under the new name:

    sudo hostname ALPHA-SRV-01
    sudo realm leave && sudo realm join -U Administrator alphalab.local

No output from either command means both succeeded. On Linux, silence is success.

---

## Step 3 - Clean Up AD Computer Objects

After rejoining, ALPHA-DC01 was checked to see the current state of computer objects. The result showed three entries:

- ALPHA-DC01 in OU=Domain Controllers (correct)
- UBUNTU-SERVER-0 in OU=Servers (stale -- needed to be deleted)
- ALPHA-SRV-01 in CN=Computers (correct name, wrong location)

When a machine joins a domain, Windows places the computer object in CN=Computers by default, not in the custom OU structure. This matters because GPOs linked to OU=Servers will not apply to machines sitting in CN=Computers.

Two cleanup commands were run on ALPHA-DC01:

Delete the stale object:

    Get-ADComputer -Identity "UBUNTU-SERVER-0" | Remove-ADObject -Confirm:$false

Move the new object to the correct OU:

    Get-ADComputer -Identity "ALPHA-SRV-01" | Move-ADObject -TargetPath "OU=Servers,OU=Computers,OU=AlphaLab,DC=alphalab,DC=local"

Verified with:

    Get-ADComputer -Filter * | Select-Object Name, DistinguishedName | Format-Table -AutoSize

Result:

    ALPHA-DC01    CN=ALPHA-DC01,OU=Domain Controllers,DC=alphalab,DC=local
    ALPHA-SRV-01  CN=ALPHA-SRV-01,OU=Servers,OU=Computers,OU=AlphaLab,DC=alphalab,DC=local

Clean and correct.

---

## Step 4 - Join Kali Linux to the Domain

Kali Linux uses Debian-based repos with a security tooling focus, which means some standard system integration packages are not available. The usual realm join workflow requires sssd and sssd-tools, but these are not in Kali's package repositories.

First, the available packages were installed:

    sudo apt install -y realmd adcli packagekit krb5-user samba-common-bin

The realm discover command confirmed Kali could see the domain:

    sudo realm discover alphalab.local

Output:

    alphalab.local
      type: kerberos
      realm-name: ALPHALAB.LOCAL
      domain-name: alphalab.local
      configured: no
      server-software: active-directory
      client-software: sssd

However, attempting realm join failed with:

    realm: Couldn't join realm: The following packages are not available for installation: sssd-tools sssd libnss-sss libpam-sss

The workaround was to bypass realm entirely and use adcli directly, which talks to the DC via Kerberos without requiring the full sssd stack:

    sudo adcli join alphalab.local -U Administrator --domain-controller=172.16.0.30 --show-details

Output confirmed a successful join:

    domain-name = alphalab.local
    domain-realm = ALPHALAB.LOCAL
    domain-controller = 172.16.0.30
    computer-name = KALI-LINUX
    computer-dn = CN=KALI-LINUX,CN=Computers,DC=alphalab,DC=local
    keytab = FILE:/etc/krb5.keytab

The Kerberos keytab file at /etc/krb5.keytab is what stores the machine's credentials for authenticating to the domain. This is what allows domain-aware tools to operate with trusted context.

---

## Step 5 - Move Kali into the Correct OU

Same as Ubuntu, Kali landed in CN=Computers by default. On ALPHA-DC01, it was moved to OU=Workstations since Kali is a workstation, not a server:

    Get-ADComputer -Identity "KALI-LINUX" | Move-ADObject -TargetPath "OU=Workstations,OU=Computers,OU=AlphaLab,DC=alphalab,DC=local"

Final AD computer inventory:

    ALPHA-DC01    CN=ALPHA-DC01,OU=Domain Controllers,DC=alphalab,DC=local
    ALPHA-SRV-01  CN=ALPHA-SRV-01,OU=Servers,OU=Computers,OU=AlphaLab,DC=alphalab,DC=local
    KALI-LINUX    CN=KALI-LINUX,OU=Workstations,OU=Computers,OU=AlphaLab,DC=alphalab,DC=local

Every machine is in its correct OU. GPOs linked to Servers will apply to ALPHA-SRV-01. GPOs linked to Workstations will apply to KALI-LINUX. The domain structure is working exactly as designed.

---

## DNS Is the Foundation of Everything

One important lesson from this phase: Kali initially failed to discover the domain because its DNS was pointing at OPNsense (172.16.0.1) instead of the DC (172.16.0.30). OPNsense has no knowledge of the alphalab.local zone -- that zone only exists on ALPHA-DC01.

The fix was simple -- point /etc/resolv.conf at the DC:

    nameserver 172.16.0.30

This is a core AD principle: every domain-joined machine must use the DC as its primary DNS server. Without it, Kerberos authentication cannot resolve domain names and nothing works.

---

## Snapshots Taken

| Name | Description |
|---|---|
| domain-joined-clean | Ubuntu and Kali both joined to alphalab.local, computer objects in correct OUs |

Snapshots were taken on both Ubuntu Server and Kali after confirming the clean AD state.

---

## Current Lab State

| VM | Hostname | IP | Domain Status |
|---|---|---|---|
| VM 100 | OPNsense | 172.16.0.1 | N/A (firewall) |
| VM 101 | ALPHA-SRV-01 | 172.16.0.10 | Joined - OU=Servers |
| VM 102 | KALI-LINUX | 172.16.0.20 | Joined - OU=Workstations |
| VM 103 | ALPHA-DC01 | 172.16.0.30 | Domain Controller |

---

## What Is Next

With all three VMs domain-joined and properly organized in AD, the lab is ready for the security stack phase. The next major milestone is deploying Wazuh SIEM to start collecting logs -- which will be the foundation for detecting the AD attacks we will be running in later phases.

The attack and defense phases planned after the security stack:

- Phase 8: External recon and initial access using a non-domain-joined Kali VM attacking the perimeter
- Phase 9: Internal recon and lateral movement using BloodHound, Kerberoasting, and Pass-the-Hash from within the domain
- Phase 10: Detection and response using Wazuh to catch and alert on the attacks from Phases 8 and 9

## What I Learned

- DNS is the foundation of Active Directory -- every domain-joined machine must use the DC as its primary DNS server or nothing works
- The 15-character NetBIOS name limit for computer accounts -- naming conventions must account for this from the start
- Linux silence is success -- commands like realm leave and realm join produce no output when they complete cleanly
- Computer objects always land in CN=Computers by default when joining a domain -- moving them to the correct OU is a manual step
- Why OU placement matters -- GPOs linked to OU=Servers only apply to machines that actually live in that OU
- Kali Linux repos are security-tool focused and do not ship standard system integration packages like sssd and sssd-tools
- adcli can join a machine to an AD domain without the full sssd stack -- a useful workaround for Kali specifically
- The Kerberos keytab file at /etc/krb5.keytab stores machine credentials for authenticating to the domain
- Always check the shell prompt before running commands -- running a command on the wrong machine is a real risk when managing multiple SSH sessions
- Domain-joined Kali enables authenticated AD attacks that behave differently and more powerfully than unauthenticated ones
