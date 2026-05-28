# Purple Team Simulation & Detection Engineering Lab

## 🛡️ Executive Summary
This project demonstrates a complete **Purple Team** workflow based on the **Assume Breach** model. The primary objective was to evaluate the default detection capabilities of a centralized SIEM system (Wazuh) against advanced **Living off the Land (LotL)** techniques, identify telemetry blind spots, and implement custom Detection as Code (DaC) solutions using Microsoft Sysmon.

**Authors:** Jan Wojnicki & Iwo Sidorowicz  
**Technologies:** Wazuh SIEM, Microsoft Sysmon, Windows 11, Kali Linux, MITRE ATT&CK, XML  

---

## 🏗️ Architecture & Lab Environment
The lab was built in an isolated NAT Network to ensure operational security during the execution of malicious payloads.
* **Red Team (Kali Linux):** C2 Server utilizing Netcat, Python HTTP Server, and `msfvenom` for initial access payloads.
* **Blue Team / Endpoint (Windows 11):** Stripped of perimeter defenses (Windows Defender & Firewall disabled) to allow pure behavioral log generation. Equipped with Wazuh Agent and Sysmon.
* **SIEM Central (Ubuntu):** Wazuh Manager 4.14.5 responsible for log aggregation, decoding, and correlation.

---

## ⚔️ Red Team: Attack Killchain
Assuming initial compromise, a reverse shell payload (`Antywirus_Fix.exe`) was deployed to simulate a breached endpoint. The following post-exploitation techniques were executed:

### 1. Privilege Escalation & Persistence (MITRE T1136.001)
Creation of a backdoor administrator account using native binaries:
> `net user haker_zdalny Haslo123! /add`  
> `net localgroup Administratorzy haker_zdalny /add`

### 2. OS Credential Dumping: LSASS Memory (MITRE T1003.001)
Bypassing traditional EDR signatures by utilizing the native `comsvcs.dll` library to dump the LSASS process memory:
> `rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump 824 C:\lsass_dump.dmp full`

### 3. Inhibit System Recovery: Ransomware Prep (MITRE T1490)
Simulating the early stages of a Ransomware attack by wiping local shadow copies using `vssadmin`:
> `vssadmin.exe delete shadows /all /quiet`

---

## 🛡️ Blue Team: Analysis & Detection Engineering
During the hunting phase, it was discovered that while the SIEM successfully generated Level 8 alerts for `net user` commands, it completely missed the LSASS dump and the Shadow Copy deletion. These LotL executions were categorized as benign Event Level 3 (informational).

### Closing the Blind Spot (Detection as Code)
To mitigate this, the Wazuh agent was reconfigured to ingest `Microsoft-Windows-Sysmon/Operational` event channels. Custom XML rules were engineered to flag malicious arguments within trusted processes:

1. **Rule 100050:** Detects the specific `comsvcs.dll` structure called by `rundll32.exe` mapping to T1003.001.
2. **Rule 100051:** Utilizes the `<match>` tag to catch the exact `delete shadows` string across the Windows event group, detecting Ransomware-like volume shadow manipulation regardless of the binary name used, mapping to T1490.

Upon running a re-test, the SIEM successfully generated **Level 12 (Critical)** alerts for both previously bypassed techniques.

---

## 📁 Repository Structure
* `/rules/` - Contains the custom XML detection rules developed for the Wazuh Manager.
* `/configs/` - Contains the Wazuh Agent `ossec.conf` modifications required for Sysmon ingestion.
* `/docs/` - Contains the comprehensive Polish language **Post-Incident Project Report (PDF)** detailing the methodology, troubleshooting, and full event log analysis.

> *"Security is not a product, but a process."* This repository serves as a testament to the continuous feedback loop required to defend modern infrastructure.
