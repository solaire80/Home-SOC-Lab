
# Lab 03 - SMB Authentication Detection with Wazuh

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
