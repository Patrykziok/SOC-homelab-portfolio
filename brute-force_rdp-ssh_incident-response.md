1. Lab Infrastructure
This lab environment was designed to monitor and secure a hybrid infrastructure consisting of both Windows and Linux endpoints.

Wazuh Manager: Hosted on an Ubuntu Server, acting as the central brain for log collection, correlation, and alerting.
Linux Endpoint: An Ubuntu Server running an Nginx web server, monitored by a Wazuh Agent for authentication and system logs.
Windows Endpoint: A Windows 10 workstation monitored by a Wazuh Agent and Sysmon, providing granular visibility into process creation and network connections.


2. Attack Simulation: Multi-Protocol Brute Force
To test the detection capabilities of the SIEM, I performed a dictionary-based brute force attack against both protocols: RDP (Windows) and SSH (Linux) using Hydra on a Ubuntu Linux machine.

Attack Proof (Ubuntu Linux):
Targeting: Attempting to crack the Administrator account on Windows and the soc account on Linux.
Outcome: The attack was successful, identifying the correct password (Qweqwe123) for both systems.

Linux attack command -> hydra -vV -l soc -P passwords_linux.txt 192.168.101.11 ssh
<img width="1291" height="501" alt="Hydra Atak - Linux" src="https://github.com/user-attachments/assets/731ea80c-6ad6-4c19-a129-f4253464e1a4" />

Windows attack command -> hydra -vV -l Administrator -P passwords_windows.txt 192.168.101.20 rdp
<img width="1299" height="497" alt="Hydra Atak - Windows" src="https://github.com/user-attachments/assets/ff1028f2-80ed-41f7-acb5-7e1622e43a4a" />

3. Analysis and Technical Evidence
After the attack, I analyzed the telemetry from three different perspectives to confirm the breach.
A. Wazuh SIEM Dashboard (Correlation)
The manager successfully correlated the noise from the attack into high-priority alerts.
The SIEM captured the progression from failed attempts (Level 5) to an active brute force detection (Level 10) and finally to a critical alert (Level 12).

WINDOWS: Wazuh used command for filter logs -> rule.id:(60122 OR 60204 OR 100001)
<img width="1917" height="641" alt="Hydra Windows Logi" src="https://github.com/user-attachments/assets/7c293b45-414c-4968-aaae-2db4c3664ab3" />
LINUX -> Wazuh used command for filter logs -> rule.id:(5760 OR 2502 OR 40112)
<img width="1909" height="503" alt="Hydra Linux Logi" src="https://github.com/user-attachments/assets/8ee61316-3d55-4217-af96-0db7778e6ff0" />

B. Windows Event Logs (Endpoint Side)
I verified the attack on the Windows host using the Event Viewer.
<img width="617" height="615" alt="Event log Windows 4624-4625" src="https://github.com/user-attachments/assets/394d2d4d-e1b4-4561-8c71-3e3c53b1d5de" />

Logon Failures: Multiple Event ID 4625 entries confirmed the brute force attempt.
Logon Success: A subsequent Event ID 4624 confirmed that the attacker successfully gained access.

C. Linux Auth Logs (Endpoint Side)
On the Ubuntu server, I examined the /var/log/auth.log file to trace the SSH attack.
Command used > sudo journalctl -u ssh -n 50 --no-pager
<img width="1287" height="815" alt="log auth Linux" src="https://github.com/user-attachments/assets/b44b70b9-4afb-46bd-837b-d873b0b1d553" />

Evidence: The log shows hundreds of Failed password messages from the attacker's IP, followed by an Accepted password entry for the user soc.

4. Containment: Wazuh-Driven Isolation
After confirming a successful authentication on both endpoints (Windows 4624 after 4625, Linux Accepted password after Failed password), the incident response process moved from detection to containment. The primary objective was to immediately prevent repeated access attempts and stop any potential post-compromise actions.
4.1 Isolation Actions in Wazuh (Active Response)
To isolate the affected endpoints from the attacker, I executed a Wazuh-based containment action by applying an Active Response / host-level block for the attacker’s source IP - 192.168.101.10
<img width="933" height="882" alt="Windows sucessfull login details" src="https://github.com/user-attachments/assets/9398c2e6-683f-4c57-a071-1ae70c4e743e" />

Access Control Safeguard (Temporary Account Lockdown)
Because brute force resulted in valid credentials, the incident was treated as Credential Compromise.
The affected accounts (Administrator, soc) were temporarily restricted (disable/lock) until the investigation phase completed.


5. Investigation & Threat Hunting: Did the Attacker Create a New Account?
After containment, I performed a targeted investigation to determine whether the attacker established persistence — specifically, whether a secondary account was created (a common post-compromise step).
Windows persistence evidence: Windows Security auditing recorded Event ID 4720 (User Account Management) at 2026-01-10 22:26:35, confirming creation of a new local user account ir_backdoor on host WIN-VVRDFQU4TPN. The action was performed under the Administrator context.
<img width="1019" height="771" alt="Windows konto zrobione log" src="https://github.com/user-attachments/assets/7031676c-fe9d-4b98-babb-4fe4a9fc5207" />

LINUX > sudo grep -Ei "adduser|useradd|usermod|groupadd|passwd|deluser|userdel|sudo" /var/log/auth.log | tail -n 60
I confirmed the creation of a secondary account on the Ubuntu endpoint using /var/log/auth.log. At 2026-01-10 21:50:26, privileged user management was initiated via sudo (COMMAND=/usr/sbin/adduser ir_backdoor)
<img width="1320" height="115" alt="LINUX - STWORZENIE KOONTA DOWOD" src="https://github.com/user-attachments/assets/eba8c2c4-80d4-42fd-8963-211197ea41cb" />


7. Eradication: Remove Persistence and Close the Compromise
After confirming detection capability and validating the monitoring pipeline, the response moved to eradication.

Windows:
Removed the simulated persistence account -> net user ir_backdoor /delete
Verified removal by confirming absence of the user and searching for relevant Security events:
4726 (user account deleted) may appear depending on auditing configuration.
<img width="1004" height="712" alt="Windows dowod usuniecia konta" src="https://github.com/user-attachments/assets/591ae7b5-638a-401c-b4d5-787fc999241f" />
<img width="1299" height="115" alt="LINUX - USUNIECIE KONTA DOWOD" src="https://github.com/user-attachments/assets/36df5331-1f9f-4c67-a4eb-fbab9fca9465" />


Linux:
Removed the simulated user (backup_ops) and cleaned any related artifacts: > sudo deluser --remove-home ir_backdoor
home directory, ssh keys, cron jobs if any were created.
<img width="1299" height="115" alt="LINUX - USUNIECIE KONTA DOWOD" src="https://github.com/user-attachments/assets/ad9279b6-897b-4b4c-b50f-0f032cce35e5" />



RECOMMENDATION

7. Recovery: Return Services to Normal Operation
After eradication, systems were returned to normal operations in a controlled manner.


8. Lessons Learned (SOC-Style)
Severity escalation rule: brute force becomes Critical once a success login occurs (Valid Accounts).
Containment must be immediate: IP-based blocking in Wazuh Active Response is effective to stop the attack quickly.
Persistence checks are mandatory: account creation, group membership changes, SSH key additions, and scheduled tasks must be part of every post-brute-force investigation.
Passwords are the weakest link: adopting SSH keys and lockout policies drastically reduces brute force feasibility.
Wazuh value: correlation reduces noise and provides a clear incident narrative from failed attempts → detection → compromise → response.



