
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
