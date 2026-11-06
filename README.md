# Splunk SIEM Investigation & Threat Detection Lab
---

## 1. Executive Summary
The objective of this lab is to architect a localized enterprise network, simulate common adversary tactics; specifically credential brute-forcing and phishing-initiated Command and Control (C2) as well as engineering Splunk alerts to detect these activities in real-time. By tracking the attack lifecycle from initial access to execution, this project highlights practical experience in telemetry ingestion, threat hunting, and incident response alerting.

---

## 2. Lab Architecture & Environment
The lab is built on a bare-metal Ubuntu Linux host utilizing a local hypervisor (QEMU/KVM) to isolate the attacker and victim environments, with telemetry centrally aggregated into Splunk Enterprise.

| Component | Role | Specification / Details |
| :--- | :--- | :--- |
| **SIEM / Host** | Bare-Metal Ubuntu Linux | Hosts Splunk Enterprise; manages virtualized network and VM routing. |
| **Victim Endpoint** | Windows 10 (VM) | Configured with Splunk Universal Forwarder to push local event logs. |
| **Attacker Endpoint** | Kali Linux (VM) | Offense environment utilized for payload generation and brute-force simulations. |
| **Log Transport** | Splunk Universal Forwarder | Configured on the Windows endpoint to securely forward security and operational telemetry to the Ubuntu indexer. |

### Telemetry Ingestion Setup
The Splunk Universal Forwarder was configured via `inputs.conf` to monitor and forward Windows Event Logs, including Sysmon operational telemetry. 

![Universal Forwarder inputs.conf](<Screenshots/Screenshot From 2026-11-04 21-35-49.png>)
*Universal Forwarder `inputs.conf` routing Sysmon and Security logs to the Splunk indexer.*

Raw telemetry generation was verified on the Windows 10 target via the local Event Viewer, confirming successful capture of Sysmon Event ID 1 (Process Creation) and Event ID 3 (Network Connection).

![Windows Event Viewer Sysmon Logs](<Screenshots/Screenshot From 2026-11-04 21-39-16.png>)
*Windows Event Viewer on the QEMU/KVM target confirming active Sysmon telemetry generation.*

---

## 3. Attack Simulation Methodology

### Phase 1: Credential Brute Force Simulation
* **Vector:** Initial Access / Credential Access
* **Tool Utilized:** Hydra
* **Execution:** A targeted dictionary attack was launched from the Kali Linux VM against the Windows 10 VM's SMB2 service (`192.168.122.18`) to simulate a noisy, high-volume credential stuffing attempt.

![Hydra Brute Force Execution](<Screenshots/Screenshot From 2026-11-04 21-27-29.png>)
*Hydra actively brute-forcing the Administrator account via SMB2.*

### Phase 2: Phishing & C2 Reverse Shell
* **Vector:** Execution / Command and Control (C2)
* **Tools Utilized:** Metasploit Framework (`msfvenom`, `multi/handler`)
* **Execution:** A simulated phishing scenario was executed where a weaponized executable payload (`invoice_update.exe`) was downloaded and executed on the Windows 10 VM. 

![Malicious Payload Execution](<Screenshots/Screenshot From 2026-11-04 21-29-10.png>)
*Execution of the weaponized `invoice_update.exe` payload on the target endpoint.*

The payload established an outbound reverse TCP shell proxy back to the Kali Linux listener (`192.168.122.219` on port `4444`), granting the attacker remote command-line access. Post-exploitation activity included navigating the target filesystem and exfiltrating sensitive data.

![Meterpreter Session Established](<Screenshots/Screenshot From 2026-11-04 21-30-38.png>)
*Active Meterpreter session established via the reverse TCP handler.*

![Post-Exploitation Activity](<Screenshots/Screenshot From 2026-11-04 21-31-58.png>)
*Interacting with the target filesystem through the Meterpreter reverse shell.*

---

## 4. Detection Engineering & SIEM Alerting
To provide actionable intelligence for a Security Operations Center (SOC), custom alerts were engineered in Splunk to detect the specific indicators of compromise (IoCs) generated during the attack simulations.

![Splunk Alerts Configured](<Screenshots/Screenshot From 2026-11-04 21-13-29.png>)
*Active Splunk detection alerts configured for both attack vectors.*

### Alert 1: Excessive Authentication Failures (Brute Force)
* **Detection Logic:** Monitors Windows Security Event Logs for a high threshold of failed logon attempts (Event ID 4625) originating from a single source IP.
* **Objective:** Identify and flag active brute-force or password-spraying campaigns before successful credential compromise occurs.

![Brute Force SPL Query](<Screenshots/Screenshot From 2026-11-04 21-33-59.png>)
*Splunk SPL query aggregating Event ID 4625 to detect the Hydra brute-force attack.*

### Alert 2: Suspicious Outbound Connection (Reverse Shell / Proxy)
* **Detection Logic:** Correlates process creation events with outbound network connections (Sysmon Event IDs 1 and 3) targeting non-standard ports or known suspicious external IPs.
* **Objective:** Detect the execution of the downloaded malware and the subsequent outbound C2 beaconing, alerting analysts to a potential reverse shell proxy being established.

![Reverse Shell Correlation Query](<Screenshots/Screenshot From 2026-11-04 21-33-32.png>)
*Splunk SPL query correlating Sysmon Event ID 1 and Event ID 3 to map the full execution and network callback sequence of the payload.*

### Incident Trigger Verification
Both engineered rules successfully triggered critical alerts upon payload execution and active brute-forcing, validating the end-to-end detection pipeline.

![Triggered SIEM Alerts](<Screenshots/Screenshot From 2026-11-04 21-33-14.png>)
*Splunk Triggered Alerts dashboard confirming critical detections for both simulated attacks.*

## MITRE ATT&CK Mapping

| Phase | Tactic | Technique | ID | Simulated With | Detected By | Log Source |
|---|---|---|---|---|---|---|
| 1. Credential brute force | Credential Access | Brute Force: Password Guessing | [T1110.001](https://attack.mitre.org/techniques/T1110/001/) | Hydra (SMB2, Administrator account) | **Alert 1:** Excessive Authentication Failures | Windows Security, Event ID 4625 (Logon Type 3) |
| 2. Payload execution | Execution | User Execution: Malicious File | [T1204.002](https://attack.mitre.org/techniques/T1204/002/) | Execution of `invoice_update.exe` on the Windows VM | **Alert 2:** Suspicious Outbound Connection | Sysmon, Event ID 1 (process creation) |
| 2. C2 channel | Command and Control | Non-Application Layer Protocol | [T1095](https://attack.mitre.org/techniques/T1095/) | Metasploit reverse TCP (`multi/handler`) | **Alert 2:** Suspicious Outbound Connection | Sysmon, Event ID 3 (network connection) |
| 2. C2 channel | Command and Control | Non-Standard Port | [T1571](https://attack.mitre.org/techniques/T1571/) | Callback to Kali listener on port 4444 | **Alert 2:** Suspicious Outbound Connection | Sysmon, Event ID 3 (destination port) |

### Detection Coverage

- **Covered:** T1110.001, T1204.002, T1095, T1571

---

## 5. Conclusion & Skills Demonstrated
This lab successfully models a complete attack and defense lifecycle, showcasing practical competencies in:
* **Infrastructure Setup:** Deploying and networking bare-metal and virtualized environments.
* **Log Management:** Configuring universal forwarders and ingesting endpoint telemetry.
* **Offensive Security Fundamentals:** Utilizing Hydra and Metasploit to understand attacker behavior.
* **Detection Engineering:** Translating raw log data into high-fidelity Splunk alerts for proactive threat monitoring.
