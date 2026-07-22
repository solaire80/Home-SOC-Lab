# Lab 05 - Brute Force Detection and Active Response

## Overview

**Date:** 22 July 2026

**Author:** messaoudi moncef

## Objective

The objective of this lab was to simulate a brute force authentication attack against the Windows 10 endpoint from Kali Linux using Hydra, observe how Wazuh correlates the resulting failed logon events into a high-severity brute force alert, and configure Wazuh's **Active Response** module to automatically block the attacker's IP address on the endpoint's Windows Firewall once the detection threshold is crossed — moving from detection to automated prevention without manual intervention.

---

## Lab Environment

| Component    | Description                                             |
| ------------ | ------------------------------------------------------- |
| SIEM         | Wazuh 4.12.0                                            |
| Wazuh Server | Ubuntu Server 24.04 LTS                                 |
| Endpoint     | Windows 10 Pro 22H2                                     |
| Attacker     | Kali Linux                                              |
| Hypervisor   | Oracle VirtualBox                                       |
| Logging      | Windows Security Logs + Sysmon (SwiftOnSecurity config) |

---

## Network Configuration

| Device       | IP Address     |
| ------------ | -------------- |
| Wazuh Server | 192.168.56.108 |
| Windows 10   | 192.168.56.107 |
| Kali Linux   | 192.168.56.103 |

---

## Attack Scenario

A brute force attack was launched from the Kali Linux machine against the **labuser** local account on the Windows 10 endpoint over SMB (port 445), using Hydra with the `rockyou.txt` wordlist.

Each failed authentication attempt causes Windows to log a **Security Event ID 4625** (An account failed to log on) on the endpoint. These events are forwarded to the Wazuh manager in real time via the agent. Wazuh's frequency-based correlation engine then groups the failures by source IP and, once a configurable threshold is crossed, fires a brute force detection rule.

With Active Response configured, this rule trigger is not just an alert — it becomes an instruction. The Wazuh manager sends a command back to the Windows agent, which executes a firewall script to block all inbound traffic from the attacking IP address.

This lab demonstrates the full loop: **simulate → detect → alert → block**, which reflects how a well-tuned SIEM can serve as a lightweight automated prevention layer even without a dedicated EDR or SOAR platform.

**MITRE ATT&CK:** T1110.001 – Brute Force: Password Guessing

---

## Attack Simulation

The attack was launched from the Kali Linux virtual machine using **Hydra**, targeting the SMB service on the Windows 10 endpoint:

```bash
hydra -l labuser -P /usr/share/wordlists/rockyou.txt smb://192.168.56.107 -t 1 -W 1
```

> **Note:** `-t 1` limits Hydra to one thread and `-W 1` adds a 1-second wait between attempts. This ensures each incorrect password generates a distinct, separately-logged Event ID 4625 on the Windows endpoint, which is required for Wazuh's frequency-based brute force rules to accumulate and fire correctly. Without this throttling, rapid-fire attempts can be batched or dropped by SMB negotiation before Windows logs them individually.

Hydra iterated through the wordlist, with each failed attempt returned as an authentication error:

```
[ERROR] invalid reply from target smb://192.168.56.107:445/
[445][smb] host: 192.168.56.107   login: labuser   password: 123456
[445][smb] host: 192.168.56.107   login: labuser   password: password
[445][smb] host: 192.168.56.107   login: labuser   password: 12345678
[445][smb] host: 192.168.56.107   login: labuser   password: qwerty
[445][smb] host: 192.168.56.107   login: labuser   password: abc123
[445][smb] host: 192.168.56.107   login: labuser   password: monkey
[445][smb] host: 192.168.56.107   login: labuser   password: 111111
[445][smb] host: 192.168.56.107   login: labuser   password: letmein
...
[ERROR] could not connect to target - is the service up?
```

The final connection error confirmed that Wazuh's Active Response had kicked in and the Windows Firewall rule was now blocking Hydra's connection attempts.

---

## Detection Logic

The Wazuh Security Events view, filtered by the `win10-lap` agent and the attack time window, showed a spike of Event ID 4625 alerts followed by the brute force correlation rule and the Active Response confirmation. The following rules fired in sequence:

| Rule ID | Level | Description |
| ------- | ----- | ----------- |
| 60122   | 5     | Windows logon failure (Event ID 4625 — individual failed network logon) |
| 60204   | 10    | Multiple Windows logon failures from the same source IP — possible brute force attack |
| 601     | 3     | Host blocked by Active Response — `firewall-drop` executed on agent |

**Rule 60204** is the pivotal alert. It is a frequency-based rule built into Wazuh's Windows ruleset that fires when **8 or more failed logon events** from the same source IP are observed within a **120-second window** on a single agent. No custom rule authoring was required.

Key fields visible in the Wazuh alert for rule 60204:

| Field                               | Value                         |
| ----------------------------------- | ----------------------------- |
| `data.win.eventdata.ipAddress`      | `192.168.56.103` (Kali Linux) |
| `data.win.eventdata.targetUserName` | `labuser`                     |
| `data.win.eventdata.logonType`      | `3` (Network logon)           |
| `data.win.system.eventID`           | `4625`                        |
| `rule.id`                           | `60204`                       |
| `rule.level`                        | `10`                          |

---

## Active Response Configuration

Wazuh's **Active Response** module allows the manager to push a response action to any agent when a specified rule fires. In this lab it was configured to run the built-in `firewall-drop` command on the Windows 10 agent, which calls `netsh advfirewall` to create an inbound block rule for the offending source IP.

### Step 1 — Verify the firewall-drop command exists on the Manager

The `firewall-drop` command is shipped with Wazuh by default. Confirm it is present in the manager configuration at `/var/ossec/etc/ossec.conf`:

```xml
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>
```

### Step 2 — Add the Active Response block to ossec.conf

On the **Wazuh Manager**, open the main configuration file:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following `<active-response>` block inside the `<ossec_config>` section:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>60204</rules_id>
  <timeout>300</timeout>
</active-response>
```

| Parameter      | Value          | Meaning                                                                    |
| -------------- | -------------- | -------------------------------------------------------------------------- |
| `<command>`    | `firewall-drop`| Built-in script that adds a Windows Firewall inbound block rule            |
| `<location>`   | `local`        | Execute the response on the agent that generated the alert (Windows 10)    |
| `<rules_id>`   | `60204`        | Only trigger when the brute force correlation rule fires                   |
| `<timeout>`    | `300`          | Automatically remove the block after 300 seconds (5 minutes)               |

### Step 3 — Restart the Wazuh Manager

```bash
sudo systemctl restart wazuh-manager
```

### What happens on the Windows endpoint

When rule 60204 fires, the manager sends an Active Response instruction to the `win10-lap` agent. The agent executes `firewall-drop.cmd` from `C:\Program Files (x86)\ossec-agent\active-response\bin\`, which internally runs:

```cmd
netsh advfirewall firewall add rule name="WAZUH ACTIVE RESPONSE BLOCKED" ^
  protocol=TCP dir=in action=block remoteip=192.168.56.103
```

This immediately drops all inbound TCP traffic from the Kali Linux IP. After the 300-second timeout, the agent automatically removes the rule, restoring connectivity.

---

### Figure 1 – Hydra Brute Force Attack from Kali Linux

The Kali terminal shows Hydra iterating through the `rockyou.txt` wordlist against the SMB service on the Windows 10 endpoint. Each password attempt that fails is logged as a Windows Event ID 4625 on the target. The final connection error at the bottom confirms that Wazuh's Active Response has applied the firewall block, cutting off Hydra's access.

![Hydra Brute Force Attack](screenshots/hydra-attack.png)

---

### Figure 2 – Wazuh Detection: Event ID 4625 Spike and Rule 60204 Alert

The Wazuh Security Events view shows a burst of Level 5 rule 60122 alerts (individual failed logons) followed immediately by the Level 10 rule 60204 alert — the brute force correlation. The source IP `192.168.56.103` is clearly visible across all events, linking them to the Kali machine. Rule 601 appears immediately after, confirming that Active Response executed the firewall block on the agent.

![Wazuh Detection Timeline](screenshots/wazuh-bruteforce-alerts.png)

---

### Figure 3 – Windows Firewall Rule Applied by Active Response

The Windows Firewall with Advanced Security console on the Windows 10 endpoint shows the inbound block rule created by the Wazuh Active Response script. The rule name `WAZUH ACTIVE RESPONSE BLOCKED` and the remote IP `192.168.56.103` confirm that the automated block was applied correctly in response to the brute force detection.

![Windows Firewall Rule](screenshots/firewall-rule-applied.png)

---

## Attack Timeline

1. Hydra launched from Kali Linux targeting SMB on Windows 10
2. First failed authentication attempts logged as Windows Event ID 4625
3. Wazuh agent forwards events to manager in real time
4. Rule 60122 fires on each individual failure (Level 5)
5. 8th consecutive failure within 120-second window crosses Wazuh threshold
6. Rule 60204 fires — brute force alert at Level 10
7. Wazuh manager sends Active Response instruction to `win10-lap` agent
8. Agent executes `firewall-drop.cmd` — Windows Firewall inbound block rule created for `192.168.56.103`
9. Rule 601 logged — Active Response confirmed
10. Hydra connections begin failing — IP fully blocked at the endpoint
11. After 300-second timeout, Wazuh agent removes the firewall rule automatically

---

## Analysis / Findings

This lab demonstrated a qualitative step forward compared to the previous labs — not just detecting malicious activity, but **automatically responding to it** without any human intervention.

The individual Event ID 4625 entries, while useful as raw data points, would be unmanageable at scale if a SOC analyst had to triage each one manually. Wazuh's frequency correlation (rule 60204) abstracts this into a single, actionable high-severity alert — the right signal at the right level of abstraction for a Tier 1 analyst to act on.

The Active Response configuration adds the prevention layer. Once rule 60204 fires, the time between detection and block is measured in seconds — fast enough to meaningfully limit how many password guesses an attacker can make before being cut off, and automatic enough to work even outside business hours when no analyst is actively monitoring.

The `<timeout>` parameter on the Active Response block is worth highlighting. A permanent block would risk legitimate users or services being locked out in a false positive scenario. A 5-minute automatic expiry is a pragmatic default: long enough to stop an ongoing attack session, short enough that a misconfigured monitoring tool or a single user mistyping a password repeatedly doesn't cause lasting disruption.

From a SOC analyst's perspective, the rule 60204 → rule 601 sequence in the event timeline is a clear, self-documenting audit trail showing that a threat was detected and a response was taken — exactly the kind of evidence needed for incident documentation.

---

## Detection Gaps & Improvements

- **Attacker IP rotation:** The firewall block is keyed to a specific source IP. An attacker using multiple IPs or rotating through proxies would bypass this control. Complementing Active Response with account lockout policy (e.g., 5 failed attempts → 30-minute lockout) would add a layer that is source-IP-agnostic.
- **Block scope:** The `firewall-drop` response blocks the IP only on the **endpoint** where the alert originated. If the attacker pivots and targets another machine, a new response would need to trigger on that agent separately. A network-level block at the perimeter (e.g., via a firewall API integration) would be more effective at scale.
- **Alert threshold tuning:** The default threshold of 8 failures in 120 seconds may be too permissive in a low-activity environment or too sensitive in an environment where legitimate admin scripts produce authentication noise. The `<frequency>` and `<timeframe>` values in Wazuh's Windows rules should be reviewed against the environment's baseline logon behaviour.
- **Credential success after partial brute force:** This simulation only tested the detection of failed attempts. A follow-up lab could test whether Wazuh detects a successful login that follows a burst of failures — which would represent a successful brute force and warrant a higher-severity, different response than just blocking further attempts.

---

## Conclusion

This lab confirmed that Wazuh's built-in brute force detection rules require no custom configuration to detect a SMB password spray from an external host — the Event ID 4625 pipeline established in the earlier labs is sufficient to feed the frequency correlation engine. The more significant finding was the effectiveness of the Active Response module: a four-line configuration block in `ossec.conf` was enough to turn a passive SIEM alert into an active, time-limited network block applied directly to the targeted endpoint.

Together with the detection chain from Labs 03 and 04, this lab completes a picture of how Wazuh can cover the authentication attack lifecycle: detecting a successful SMB logon (Lab 03), detecting lateral movement via remote service execution (Lab 04), and now detecting and actively disrupting a credential guessing attack before it succeeds (Lab 05).
