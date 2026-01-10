## Table of Contents
1. [Lab Infrastructure](#1-lab-infrastructure)
2. [Attack Simulation](#2-attack-simulation)
3. [Analysis and Technical Evidence](#3-analysis-and-technical-evidence)
4. [Containment: Wazuh-Driven Isolation](#4-containment-wazuh-driven-isolation)
5. [Investigation & Threat Hunting](#5-investigation--threat-hunting)
6. [Eradication](#6-eradication)
7. [Recovery](#7-recovery)
8. [Lessons Learned](#8-lessons-learned)
9. [Incidient report](#9-incident-report)

# This lab environment was designed to monitor and secure a hybrid infrastructure consisting of both Windows and Linux endpoints.

## 1. Lab Infrastructure:
Wazuh Manager (SIEM): Central server responsible for log analysis, event correlation, and real-time alerting. <br>
Linux Endpoint (Ubuntu + Nginx): A web server monitored for both system authentication (SSH) and web application logs. <br>
Windows Endpoint (Win 10 + Sysmon): A workstation providing deep visibility into login events and system process activity. <br>


## 2. Attack Simulation: Multi-Protocol Brute Force
To test how well the SIEM detects real-world threats, I launched a dictionary-based brute force attack using Hydra from an Ubuntu machine. I targeted two common entry points - RDP on Windows and SSH on Linux.

Attack Proof (Ubuntu Linux):<br>
Targets: I attempted to crack the Administrator (Windows) and soc (Linux) accounts.<br>
Outcome: The attack was successful, identifying the correct password (Qweqwe123) for both systems.<br>

### Linux attack command -> hydra -vV -l soc -P passwords_linux.txt 192.168.101.11 ssh
<img width="1291" height="501" alt="Hydra Atak - Linux" src="https://github.com/user-attachments/assets/731ea80c-6ad6-4c19-a129-f4253464e1a4" />

### Windows attack command ->  hydra -vV -l Administrator -P passwords_windows.txt 192.168.101.20 rdp 
<img width="1299" height="497" alt="Hydra Atak - Windows" src="https://github.com/user-attachments/assets/ff1028f2-80ed-41f7-acb5-7e1622e43a4a" />

## 3. Analysis and Technical Evidence <br>
After the attack, I analyzed the telemetry from three different perspectives to confirm the breach.
## A. Wazuh SIEM Dashboard
The SIEM captured the progression from failed attempts (Level 5) to an active brute force detection (Level 10) and finally to a critical alert (Level 12).

### WINDOWS: Wazuh used command for filter logs -> rule.id:(60122 OR 60204 OR 100001)<br>
<img width="1917" height="641" alt="Hydra Windows Logi" src="https://github.com/user-attachments/assets/7c293b45-414c-4968-aaae-2db4c3664ab3" /> <br>
### LINUX -> Wazuh used command for filter logs -> rule.id:(5760 OR 2502 OR 40112)<br>
<img width="1909" height="503" alt="Hydra Linux Logi" src="https://github.com/user-attachments/assets/8ee61316-3d55-4217-af96-0db7778e6ff0" /> <br>

## B. Windows Event Logs - Endpoint Side
I verified the attack on the Windows host using the Event Viewer.
<img width="617" height="615" alt="Event log Windows 4624-4625" src="https://github.com/user-attachments/assets/394d2d4d-e1b4-4561-8c71-3e3c53b1d5de" /> <br>

Logon Failures: Multiple Event ID 4625 entries confirmed the brute force attempt.<br>
Logon Success: Event ID 4624 confirms a successful login right after the failed attempts.<br>

## C. Linux Auth Logs - Endpoint Side
On the Ubuntu server, I examined the /var/log/auth.log file to trace the SSH attack.<br>
Command used > sudo journalctl -u ssh -n 50 --no-pager<br>
<img width="1287" height="815" alt="log auth Linux" src="https://github.com/user-attachments/assets/b44b70b9-4afb-46bd-837b-d873b0b1d553" />

Evidence: The logs show a series of failed login attempts, followed by a successful login for the user 'soc

## 4. Containment: Wazuh-Driven Isolation
Once I confirmed the successful logins on both systems, I moved from detection to containment. My main goal was to block the attacker immediately and prevent any further damage. <br>
## 4.1 Isolation Actions in Wazuh (Active Response)<br>
To isolate the affected endpoints from the attacker, I executed a Wazuh-based containment action by applying an Active Response / host-level block for the attacker’s source IP - 192.168.101.10<br>
<img width="933" height="882" alt="Windows sucessfull login details" src="https://github.com/user-attachments/assets/9398c2e6-683f-4c57-a071-1ae70c4e743e" />

Since the attacker cracked the passwords, I treated this as a Credential Compromise. I immediately locked the 'Administrator' and 'soc' accounts to stay safe while I finished the investigation. <br>

## 5. Investigation & Threat Hunting: Did the Attacker Create a New Account?
After containing the threat, I checked if the attacker tried to maintain access by creating a new account, which is a common tactic after a breach. My investigation found the following evidence on Windows: Windows Security auditing recorded Event ID 4720 (User Account Management) at 2026-01-10 22:26:35, confirming creation of a new local user account ir_backdoor on host WIN-VVRDFQU4TPN. The action was performed under the Administrator context <br>
<img width="1019" height="771" alt="Windows konto zrobione log" src="https://github.com/user-attachments/assets/7031676c-fe9d-4b98-babb-4fe4a9fc5207" />

### LINUX > sudo grep -Ei "adduser|useradd|usermod|groupadd|passwd|deluser|userdel|sudo" /var/log/auth.log | tail -n 60 <br>
I confirmed that a new local user account ir_backdoor was created on the Ubuntu endpoint via sudo adduser. Evidence: /var/log/auth.log shows COMMAND=/usr/sbin/adduser ir_backdoor at 2026-01-10 21:50:26. <br>
<img width="1320" height="115" alt="LINUX - STWORZENIE KOONTA DOWOD" src="https://github.com/user-attachments/assets/eba8c2c4-80d4-42fd-8963-211197ea41cb" />


## 6. Eradication: Remove Persistence and Close the Compromise
Once I confirmed detection capabilities and validated the monitoring pipeline, I proceeded with eradication. <br>

### Windows:<br>
Persistence Removal: I deleted the simulated backdoor account using: net user ir_backdoor /delete.<br>
Verification: I confirmed the account's removal and audited the Security logs for Event ID 4726 (User account deleted) to ensure the action was properly recorded.<br>
<img width="1004" height="712" alt="Windows dowod usuniecia konta" src="https://github.com/user-attachments/assets/591ae7b5-638a-401c-b4d5-787fc999241f" />
<img width="1299" height="115" alt="LINUX - USUNIECIE KONTA DOWOD" src="https://github.com/user-attachments/assets/36df5331-1f9f-4c67-a4eb-fbab9fca9465" />


### Linux:<br>
User Removal: I deleted the simulated backdoor account and purged its associated data: sudo deluser --remove-home ir_backdoor<br>
<img width="1299" height="115" alt="LINUX - USUNIECIE KONTA DOWOD" src="https://github.com/user-attachments/assets/ad9279b6-897b-4b4c-b50f-0f032cce35e5" />


## 7. Recovery: Return Services to Normal Operation
After eradication, systems were returned to normal operations in a controlled manner.


## 8. Lessons Learned 
Visibility: Monitoring is essential to understand how a breach occurred. <br>
Speed: Automated blocking is the most effective way to limit the damage. <br>
Priority: A successful login during a brute-force attack must be treated as a Critical alert. <br>
Persistence: Checking for "backdoors" (like new accounts) is mandatory after any successful breach. <br>



## 9. Incidient report <br>
### Incident Overview <br>
Incident name: Multi-Protocol Brute Force with Successful Authentication (SSH + RDP) <br>
Severity: Critical (successful login confirmed) <br>
Detection platform: Wazuh SIEM/XDR (agents + correlation) <br>
Date: 2026-01-10 <br>
Impacted assets: <br>
Windows endpoint: WIN-VVRDFQU4TPN (agent.ip: 192.168.101.20) <br>
Linux endpoint: wazuh-agent-1 (192.168.101.11, SSH/Nginx) <br>
Attacker IOC (source IP): 192.168.101.10 (confirmed via Wazuh field data.win.eventdata.ipAddress) <br>
Compromised accounts: Administrator (Windows), soc (Linux) <br>
Initial access method: Dictionary brute force via Hydra <br> 
Status: Contained → Eradicated → Recovered → Closed <br>

