---
title: "Debug Log - Wazuh Agent Won't Stay Enrolled"
weight: 1
draft: false
---

# Debug Log - Wazuh Agent Won't Stay Enrolled 🔴

**Phase:** 7b - Wazuh SIEM
**Date:** June 17-18, 2026
**Status:** Resolved

---

## The Symptom

After installing Wazuh agents on ALPHA-DC01, ALPHA-SRV-01, and KALI-LINUX, the dashboard showed agents connecting briefly then dropping to Disconnected -- or worse, not appearing in the Endpoints list at all despite the agent service running on the VM.

The manager logs showed connections being rejected. The agents showed as registered locally but the dashboard either didn't reflect them or showed them as never having connected.

---

## Initial Theory - Key Mismatch

Wazuh uses a shared key system for agent authentication. When an agent is enrolled, the manager generates a unique key for that agent. That key is stored on both the manager and the agent, and every communication between them is authenticated using it.

The working theory was that the keys had become mismatched -- the manager had one version, the agent had another, and they were refusing to talk to each other as a result. This can happen if an agent is reinstalled without properly deregistering it first, or if the manager state and the agent state get out of sync.

The prescribed fix for a key mismatch is to re-import the key on the agent side using the `agent-auth` tool, or to manually re-enter the key via `manage_agents`. Both approaches were attempted.

---

## Step 1 - Attempting to Re-Import the Key

The agent key for the affected VM was pulled from the manager:

    sudo /var/ossec/bin/manage_agents

Inside the manage_agents utility, option `e` (extract key) was used to get the key string for the affected agent. That key was then taken to the agent VM and imported:

    sudo /var/ossec/bin/manage_agents

Then option `i` (import key) and the key string pasted in.

The agent service was restarted after the import:

    sudo systemctl restart wazuh-agent

Status check:

    sudo systemctl status wazuh-agent

The service showed active (running). The dashboard... still showed the agent as disconnected or pending.

---

## Step 2 - Verifying the Key Existed

The next step was confirming the key had actually been written:

    sudo cat /var/ossec/etc/client.keys

The file was empty. Or it existed but had no content. The key import had appeared to succeed but nothing was actually written.

This was the point where the diagnosis shifted from "key mismatch" to "something more fundamentally broken in the agent installation state."

---

## Step 3 - Attempting Key Reinstall

The next attempt was uninstalling and reinstalling just the agent package, hoping a clean reinstall would fix whatever was corrupted in the local agent state:

    sudo dpkg --purge wazuh-agent
    sudo rm -rf /var/ossec

Then reinstalling from scratch:

    sudo WAZUH_MANAGER='172.16.0.50' WAZUH_AGENT_NAME='ALPHA-SRV-01' dpkg -i wazuh-agent_4.9.2-1_amd64.deb

The `client.keys` file was still not populating after the reinstall. The enrollment environment variables (`WAZUH_MANAGER` and `WAZUH_AGENT_NAME`) passed to dpkg are supposed to auto-register the agent with the manager during install, but the registration wasn't completing.

---

## The Real Problem - Stale Agent Registration on the Manager

After checking the manager side, the root cause became clear. The agent had been registered with the manager during the first install -- it had an entry in the manager's agent list with an assigned ID. When the agent was reinstalled, it tried to register again under the same name, and the manager rejected or confused the duplicate registration.

The manager had a ghost entry: an agent ID associated with ALPHA-SRV-01 that was connected to old key material that no longer matched anything on the agent side. The fresh install was trying to auto-register, but the manager was either returning the old key (which the agent then couldn't verify) or blocking the new registration entirely.

---

## Resolution - Full Removal and Clean Re-Enrollment

The fix was to remove the agent completely from the manager side first, then do a clean enrollment:

**On the manager VM (wazuh-siem):**

    sudo /var/ossec/bin/manage_agents

Option `r` (remove agent) -- removed the stale entry for the affected agent entirely.

**On the agent VM:**

    sudo systemctl stop wazuh-agent
    sudo dpkg --purge wazuh-agent
    sudo rm -rf /var/ossec
    sudo rm -f /etc/systemd/system/wazuh-agent.service
    sudo systemctl daemon-reload

Reinstall with manager enrollment variables:

    sudo WAZUH_MANAGER='172.16.0.50' WAZUH_AGENT_NAME='ALPHA-SRV-01' dpkg -i wazuh-agent_4.9.2-1_amd64.deb
    sudo systemctl enable wazuh-agent
    sudo systemctl start wazuh-agent

**Verification:**

    sudo cat /var/ossec/etc/client.keys

This time the file had content -- a properly formatted key entry. The agent showed up in the Wazuh dashboard within about 30 seconds, status green, active.

The same procedure was applied to any other agent that had gotten into the same broken state.

---

## Snapshots After Resolution

Once all three agents were confirmed active and the dashboard showed green across the board, a snapshot was taken immediately:

- **`wazuh-3-agents-active`** -- all three agents enrolled and active, clean known-good state

This is the snapshot that would be used to roll back if anything breaks in the next phase of lab work. The lesson from this troubleshooting session reinforced a principle from Phase 6b: taking a snapshot of a half-broken state is not the same as taking one after a confirmed working state. The post-verification snapshot is the one that actually matters.

---

## Why This Happens - The Underlying Pattern

The Wazuh manager tracks agent identity by ID, not just by name. When an agent with ID 002 registers a key, that key is tied to ID 002 in the manager's database. If the agent is wiped and reinstalled without first removing it from the manager, the manager still has the old ID 002 entry with old key material.

The new agent install generates fresh key material and tries to register -- but the manager already has an entry for that name or ID. Depending on the Wazuh version and configuration, it may return the old key, reject the registration, or create a duplicate entry with a new ID that the agent doesn't know about.

The correct deregistration order is always: **remove from manager first, then remove from agent**. Doing it in the wrong order is what creates the ghost entry.

This same pattern shows up in certificate-based authentication systems across the board -- Puppet, Chef, mutual TLS, and others all have variations of this "stale registration" failure mode. Understanding it in Wazuh makes the pattern recognizable everywhere else.

---

## Resolution Checklist (for Future Reference)

If a Wazuh agent is showing Disconnected or Never Connected despite the service running:

1. Check `sudo cat /var/ossec/etc/client.keys` on the agent -- if empty, the enrollment failed
2. On the manager, run `manage_agents` and check if a stale entry exists for that agent name
3. Remove the stale entry from the manager (`r` in manage_agents)
4. On the agent: stop service, purge package, delete `/var/ossec`, reload systemd
5. Reinstall the agent package with `WAZUH_MANAGER` and `WAZUH_AGENT_NAME` environment variables
6. Start and enable the service, verify `client.keys` has content
7. Confirm green status in the dashboard before taking a snapshot
