# 🛡️ Home SOC Lab

A hands-on Security Operations Center (SOC) lab built to simulate real-world attack scenarios and investigate them using the Wazuh SIEM platform.

The goal of this project is to strengthen my blue team skills by building a detection-focused home lab, performing attack simulations, and documenting the investigation process as if I were working in a real SOC.

Each lab folder contains a full write-up simulating an attack technique and documenting the investigation process a SOC analyst would follow.

👉 Start here:

---

## 🎯 Objectives

- Build a functional SOC lab using Wazuh
- Simulate common attacker techniques
- Investigate security events from a defender's perspective
- Improve detection engineering and log analysis skills
- Create professional documentation for each lab

---

## 🖥️ Lab Architecture

Kali Linux
          (192.168.56.103)
                 │
      Attack Simulation
                 │
                 ▼
    Windows 10 Endpoint
      (192.168.56.107)
                 │
         Wazuh Agent
                 │
                 ▼
     Wazuh Server (Ubuntu)
      (192.168.56.108)

      ## Technologies Used

| Technology | Version |
|------------|---------|
| Wazuh | 4.12.0 |
| Ubuntu Server | 22.05.5 LTS |
| Windows 10 Pro | 22H2 |
| Kali Linux | 2025.2 |
| Sysmon | 15.15 |
| Oracle VirtualBox | 7.1.12 |

---

# 📂 Labs

| Lab | Description | Status |
|------|-------------|:------:|
| 01 | [Wazuh Installation](Labs/01-Wazuh-Installation) | ✅ |
| 02 | [Sysmon Integration](Labs/02-Sysmon-Integration) | ✅ |
| 03 | [SMB Authentication Detection](Labs/03-SMB-Authentication-Detection) | ✅ |
| 04 | [PsExec Execution Detection](Labs/04-PsExec-Execution) | ✅ |

---

## 📖 What You'll Find

Each lab includes:
- Attack scenario
- Lab environment
- Attack simulation
- Detection logic
- Evidence and screenshots
- Analysis and findings
- MITRE ATT&CK mapping
- Lessons learned

---

## 🚀 Skills Demonstrated

- SIEM Administration
- Log Analysis
- Windows Event Investigation
- Threat Hunting
- Incident Analysis
- MITRE ATT&CK Mapping
- Detection Engineering
- Security Documentation

---

> This repository documents my journey of learning Security Operations, detection engineering, and threat hunting through practical, hands-on labs.
