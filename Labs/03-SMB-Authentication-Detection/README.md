
# Lab 03 - SMB Authentication Detection with Wazuh

## Overview

**Date:** 22 July 2026

**Author:** messaoudi moncef

## Objective

The objective of this lab was to simulate a successful SMB authentication attack against a Windows 10 endpoint using the Metasploit Framework and verify that Wazuh successfully detected and analyzed the generated Windows Security Events.

---

## Lab Environment

| Component | Description |
|-----------|-------------|
| SIEM | Wazuh |
| Wazuh Server | Ubuntu Server |
| Endpoint | Windows 10 |
| Attacker | Kali Linux |
| Hypervisor | Oracle VirtualBox |
| Logging | Windows Security Logs + Sysmon |

---

## Network Configuration

| Device | IP Address |
|---------|------------|
| Wazuh Server | 192.168.56.108 |
| Windows 10 | 192.168.56.107 |
| Kali Linux | 192.168.56.103 |

---

# Attack Scenario

A Windows 10 endpoint was configured with a local administrator account named **labuser** to simulate a legitimate enterprise user.

From the Kali Linux virtual machine, the Metasploit Framework was used to perform a successful SMB authentication against the Windows endpoint.

The objective was to determine whether Wazuh would successfully collect the generated Windows Security Events and alert on the remote authentication activity.

Although the authentication used legitimate credentials, this technique is commonly observed during lateral movement when an attacker gains access to valid user accounts inside a network.

## Attack Simulation / Setup

The attack was performed from the Kali Linux virtual machine using the Metasploit Framework. Before launching the attack, a local administrator account named **labuser** was created on the Windows 10 endpoint. This account was used because Windows Microsoft accounts do not authenticate over SMB in the same way as local accounts, making a local account more suitable for this simulation.

The Metasploit Framework was started and the SMB login scanner module was selected.

**Metasploit Module**

```text
auxiliary/scanner/smb/smb_login
```

The following commands were used during the simulation:

```bash
msfconsole

use auxiliary/scanner/smb/smb_login

set RHOSTS 192.168.56.107
set SMBUser labuser
set SMBPass Lab123!

run
```

After the module was executed, Metasploit successfully authenticated to the Windows endpoint using the supplied credentials. This generated Windows Security Events that were later analyzed using Wazuh.

## Detection Logic

After executing the SMB authentication attack, I reviewed the generated alerts in the Wazuh **Threat Hunting** module to verify that the activity had been detected.

The Windows endpoint generated several security events that were processed by the Wazuh Manager. The authentication sequence produced multiple alerts, allowing the login session to be reconstructed.

The following alerts were observed during the investigation:

| Rule ID | Alert | Level |
|---------:|-------|------:|
| 92657 | Successful Remote Logon Detected (NTLM Authentication) | 6 |
| 67028 | Special privileges assigned to new logon | 3 |
| 60137 | Windows User Logoff | 3 |

These alerts confirmed that Wazuh successfully detected the remote SMB authentication performed using the **labuser** account. The correlated alerts also showed that administrative privileges were assigned to the session and that the logon sequence completed normally before the user logged off.

### Figure 1 – Successful SMB Authentication Using Metasploit

The Metasploit `auxiliary/scanner/smb/smb_login` module successfully authenticated to the Windows 10 endpoint using the **labuser** account. This confirmed that the credentials were valid and generated the authentication events required for the detection exercise.

> *(screenshots/metasploit.png)*

---

### Figure 2 – Wazuh Threat Hunting Results

The Threat Hunting module displayed the alerts generated during the SMB authentication. Wazuh detected the successful remote logon, the assignment of special privileges, and the subsequent user logoff.

> *(screenshots/)*

---

### Figure 3 – Successful Remote Logon Alert (Rule ID: 92657)

This alert identified the successful NTLM remote authentication performed by the **labuser** account and classified the activity as a successful remote logon.

> *(screenshots/rule-92657)*

---

### Figure 4 – Special Privileges Assigned (Rule ID: 67028)

This alert showed that elevated privileges were assigned to the authenticated session after the successful logon.

> *(screenshots/rule-67028)*

---

### Figure 5 – Windows User Logoff (Rule ID: 60137)

This alert confirmed that the authenticated session ended normally after the SMB authentication was completed.

> *(screenshots/rule-60137)*

